# Repraft — Architecture

High-level design of the review-response pipeline. This document describes
*what the system guarantees and why*, not line-by-line internals. Identifiers,
secrets, and environment-specific configuration are deliberately omitted.

---

## 1. Design posture

Three assumptions drive every structural decision:

- **All inbound text is hostile.** Review text is attacker-controlled input
  that lands in a model prompt. It is sanitized, scanned, and redacted before
  it is stored, and it is framed to the model as untrusted data rather than
  instruction.
- **The model is not trusted to be safe.** Model output is checked by
  deterministic code before it can be published. A guardrail that depends on an
  LLM to decide whether LLM output is safe fails exactly when the model is
  already misbehaving, so the always-on baseline is regex and rule based. A
  narrower semantic check now runs in addition to that baseline, never in
  place of it.
- **Failure must degrade to "a human looks at it."** Every error path —
  provider outage, node exception, guardrail fault, persistence failure —
  terminates in a flagged or error state that a human picks up. No failure path
  reaches "approved."

The corollary is that **approval is the exception path**. The system's default
answer is to escalate.

---

## 2. Request lifecycle

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  HTTP request                                                    │
  │    correlation id assigned → rate limits → tenant auth           │
  └───────────────────────────────┬──────────────────────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │  Validation + sanitation  │   Pydantic model:
                    │  (schema layer)           │   HTML/script strip,
                    └─────────────┬─────────────┘   hostile-keyword scan
                                  │
                    ┌─────────────▼─────────────┐
                    │  PII gatekeeper           │   the ONE place that masks
                    │  (single choke point)     │   returns (clean text,
                    └─────────────┬─────────────┘    fired categories)
                                  │
                    ┌─────────────▼─────────────┐
                    │  Persist `pending` review │   idempotent per
                    │  → return 202 to caller   │   (tenant, upstream id)
                    └─────────────┬─────────────┘
                                  │  background task — model work is
                                  │  never on the request path
                    ┌─────────────▼─────────────┐
                    │  LangGraph pipeline       │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │  Persist terminal state   │
                    │  + notify (best effort)   │
                    └───────────────────────────┘
```

The response to the caller is returned **before** any model work begins. The
aggregator gets a fast `202`; the expensive, failure-prone work runs detached,
where a failure becomes a dead-letter row rather than a webhook timeout that
triggers an upstream retry.

### Idempotency

Ingestion is idempotent on `(tenant, upstream review id)`. A pre-insert lookup
short-circuits a repeat delivery, and a **database unique constraint** is the
backstop for the race where two duplicates pass that lookup simultaneously; the
losing insert is caught, rolled back, and resolved to the winning row. Without
this, a network blip that fires the same webhook three times would create three
records and pay for three model runs. Constraint-first rather than
check-first is deliberate: the application-level lookup is an optimization, the
constraint is the guarantee.

---

## 3. The sanitation boundary

PII redaction happens at **exactly one place**: an ingestion-boundary
gatekeeper module that wraps the masking implementation and returns both the
cleaned text and the set of PII categories that fired.

This is a deliberate architectural choice, and the non-obvious part is that
**downstream layers are forbidden from re-masking**. Defense-in-depth re-masking
sounds safer but is worse here:

- With N masking call sites, "is this string clean?" has N answers and the
  invariant becomes untestable.
- Every extra call site is a place where someone adds a code path that *skips*
  it, and nothing detects the omission.

With one gatekeeper, the contract is checkable: text entering the system passes
through it, and every consumer downstream may assume clean input. The stored
record therefore holds post-redaction content — customer PII never reaches the
model provider, the database, or an operator's screen.

The categories that fired are carried forward as *metadata* (category names
only, never values) so the guardrail can escalate a review that **carried** PII
even when the resulting draft is clean. A public reply touching a
customer-reported PII situation belongs with a human regardless of how clean the
draft reads.

**Coverage.** Structured identifiers — contact details, government and
financial identifiers, network addresses, record and account numbers, dates of
birth, and similar — are matched by pattern. Personal-name detection is
explicitly **high-precision / low-recall**: a name is masked only when it
follows an explicit lexical trigger (a self-identification phrase, a labelled
field, an honorific, a sign-off). Arbitrary names are not regex-detectable
without NER, and a greedy heuristic that redacts any capitalized word destroys
ordinary review text. This limitation is documented rather than papered over;
full NER is the recall follow-up.

Masking is **idempotent** — already-redacted output contains no live patterns
and passes through a second call unchanged. That property is what makes the
gatekeeper safe to re-enter on a replay or retry.

### Logs

Structured logs pass through a masking processor positioned after
context-binding and before rendering, so every log line — including foreign
library logs — is masked uniformly from one place before it reaches stdout or
any external aggregator. This backs up the primary rule that call sites log
non-PII fields only (ids, counts, exception class names). Notably, error paths
log the *exception class*, never the stringified exception, because a stringified
error can quote the very text that triggered it.

---

## 4. The agent pipeline

A LangGraph state machine with deterministic entry and exit stages wrapped
around the model calls:

```
  START
    │
    ▼
  intake        deterministic — HTML strip, hostile-keyword scan
    │           (no model call: an attacker must not be able to
    │            pivot the sanitizer itself)
    ▼
  sentiment     model call — conservative three-way classification
    │
    ▼
  context       deterministic — select the tenant's guideline for
    │           this sentiment; fall back to safe defaults
    ▼
  drafter   ◀──────────────┐   model call — untrusted review framed
    │                      │   explicitly as data, not instruction
    ▼                      │
  guardrail ───────────────┘   deterministic — bounded refinement loop
    │
    ├──▶ draft ok      → response persisted
    ├──▶ flagged       → human review queue
    ├──▶ flagged (PII) → operator security alert, draft never published
    └──▶ system error  → dead-letter row, retryable
```

**Every node except the guardrail is wrapped in a failure decorator** that
converts an exception into a terminal state. A provider outage becomes a
retryable system-error terminal; an unexpected bug becomes a flagged review.
Either way the worker survives and the review is never silently stranded in
`pending`.

The **guardrail is deliberately not wrapped**. Wrapping it would mean an
exception inside the guardrail is handled by generic machinery, and the one
piece of code whose entire job is to block must own its own failure semantics.
Instead it is fail-closed by construction: its detector sweep runs under a
guard, a `detection_complete` flag flips only after every detector has run, and
the single pass branch is gated on that flag **plus** a zero-violation result. A
detector that faults part-way records an escalating error flag and routes to
flagged — it can neither reach pass nor propagate out of the node. This matters
because the guardrail is also invoked *directly* by the human-approval path,
outside the graph's failure machinery.

### What the guardrail checks

On the **input** side: prompt-injection patterns (instruction override, role
hijacking, prompt-leak and data-enumeration attempts, special-token and
delimiter injection, jailbreak labels) plus a broader hostile-keyword signal
shared with the schema layer.

On the **output** side: liability admissions, AI self-identification,
discounts or codes the tenant's guideline never authorized, empty drafts, and
an independent PII sweep of the generated text.

Plus an **instructional-persistence** check: when the input was already flagged
hostile, did the draft echo or obey the attacker's framing? This catches the
case where the drafter was successfully steered and the output looks superficially
well-formed.

### Two failure tiers

- **Refinable** violations (liability, AI self-identification, unauthorized
  discount, empty draft) return to the drafter with a specific corrective
  instruction and the rejected draft, for a small bounded number of rewrites.
  Concrete "you did X, do Y instead" feedback outperforms a vague retry —
  the model needs an explicit delta.
- **Hard fails** (hostile input, PII in input or draft, instructional
  persistence, guardrail fault) skip refinement entirely and escalate. A rewrite
  cannot neutralize an attack already sitting in the model's context, so
  looping only spends tokens on a foregone conclusion.

When the refinement budget is exhausted, the run is flagged with an escalation
marker that raises the human notification to its high-urgency form.

### Draft handling at the terminals

Published drafts carry a fixed **AI-disclosure suffix**, appended *after* the
guardrail has run on the bare draft — so the disclosure can never be mistaken
for AI self-identification by the guardrail's own detectors, and a refinement
pass cannot strip it. The append is idempotent, so a checkpoint replay cannot
double-stamp it.

A PII-leaking draft gets its own terminal: it is re-masked before the state is
persisted (so anything that later reads that state sees redaction tokens, not
raw PII), no response record is written, and the notification carries **no
draft or review text at all** — only identifiers and flags. It is structurally
unpublishable; the approval path rejects it outright.

---

## 5. Data model

Four tables, deliberately small.

| Entity | Role | Notes |
|---|---|---|
| **Tenant** | Customer account | Stores only the **hash** of the API key, never the plaintext. Holds a JSON config blob (per-sentiment reply guidelines, usage tier, per-provider webhook secrets) and an operational lifecycle status. |
| **Review** | Inbound review | Holds post-redaction text by contract. Carries the upstream identifier used for idempotency, unique per tenant. |
| **Response** | Generated reply | Tenant id is denormalized onto the row so the uniform tenant filter applies directly — isolation never depends on join-side filtering. |
| **FailedTask** | Dead-letter row | One open row per failed review, with retry count and resolution timestamp. |

Every tenant-scoped table carries an indexed tenant foreign key via a shared
mixin, which is also the marker the repository layer type-checks against.

Per-tenant provider secrets live in the tenant's config rather than in global
configuration, so rotating one tenant's secret never touches another's.

Schema changes are versioned migrations applied at container start.

---

## 6. Persistence and state

Two connection pools open at application startup and share one database:

- An **async ORM engine** for application queries, with pre-ping enabled so a
  connection that died during a database restart is recycled rather than
  handed to a request.
- A **separate driver pool** backing the agent framework's state checkpointer.
  Two pools rather than one is not redundancy — the checkpointer requires a
  different PostgreSQL driver than the ORM uses, so sharing is not possible.
  Its checkout timeout is sized to absorb webhook bursts that would otherwise
  surface as pool-timeout errors.

The graph is compiled at startup rather than import time so the checkpointer
draws a connection per execution instead of holding one open for the process
lifetime.

---

## 7. The six invariants

### 1. Tenant isolation

A generic **tenant-scoped repository** is the only sanctioned data-access path.
Its constructor requires a tenant id and refuses to build without one, and it
verifies at construction that the model is tenant-scoped — so there is no silent
path that bypasses the filter. Every read, update, and delete has the tenant
predicate bound into the statement. Creates **drop any caller-supplied tenant
id** and force the repository's own: the repository, not the request body, is
the authority on who owns a row.

Denormalized tenant columns create a subtler hole — a row that *claims* tenant B
while pointing at tenant A's parent. Repositories therefore declare their
foreign-key parents, and creates verify parent ownership within the same
transaction before inserting, raising a permission-typed error on mismatch.

Handlers never accept a tenant identifier from the client; identity is resolved
from the credential, so there is no parameter to tamper with.

There is exactly **one** sanctioned cross-tenant read — a platform benchmark —
isolated in a single explicitly-named module so cross-tenant access is
greppable to one file rather than smuggled inline into a per-tenant handler.
It takes no tenant parameter (so a caller cannot narrow a platform aggregate
down to one tenant) and enforces a **minimum-cohort floor**, suppressing any
aggregate computed over too few tenants. Below that floor, an aggregate leaks
individual data by differencing: if the "platform average" covers a handful of
tenants and you are one of them, you can back out the others.

### 2. Single PII gatekeeper

One module masks PII, at the ingestion boundary, before storage, before the
model, before logs. Everything downstream trusts the contract and does not
re-mask. See §3 for why one choke point beats several.

The exit-stage guardrail's independent PII sweep of the *draft* is not a
violation of this rule — it is a **detector**, not a masker, covering a
different threat: PII the model hallucinated or echoed, which the input
gatekeeper never saw.

### 3. Guardrail fails closed

The guardrail can only block, never approve under uncertainty. Its outcome
defaults to BLOCK; the single pass branch requires a fully-completed,
zero-violation sweep across both the deterministic detectors and the
semantic layer that runs after them. Node failures and provider outages
route to flagged or system-error terminals — an error inside the semantic
layer is one such fault: it blocks the draft rather than passing it, and is
logged as a distinct event from an actual detection so a semantic-layer
outage is never mistaken for a clean pass. No exception path reaches
approved. See §4.

### 4. Human edits are re-guarded

The human-in-the-loop path is where a well-meaning operator can undo every
upstream control. An operator may edit a draft before approving it — and the
edited text is re-run through the **same** guardrail before the approval is
allowed to proceed. An edit that reintroduces a liability admission or PII is
rejected and the review stays flagged.

Refinement is **disabled** for this re-check: a violation hard-fails rather than
looping back into the model. The operator owns that text, so the shield's job is
to block it, not to silently rewrite a human's words and publish the result.

Both approval flavors — a signed click-through link and an authenticated API
call — share one stale-state contract, rejecting approval of a review that was
already responded to or that has aged past the response window. Late replies
read as tone-deaf on the upstream platform, and a silent success on a re-click
hides that the reply already went out.

### 5. Kill switch — ordering *and* enforcement

Two distinct guarantees, both required. Either one alone is insufficient: a halt
that isn't enforced doesn't stop anything, and an enforced halt that was
announced before it persisted misleads the operator who acts on the alert.

**Ordering.** An operator kill switch flips a tenant's status and alerts the
operator channel. The database is the source of truth, so it must reflect the
halt *before* anyone is told the halt happened. The commit physically precedes
the notification call, so a commit failure raises before the notification can
fire — the system never announces a halt that did not persist. Re-killing an
already-halted tenant is an idempotent no-op with no duplicate alert, and the
notification channel being unavailable never blocks the halt itself.

**Enforcement.** The halted status is read on the request path: tenant
authentication resolves the credential to a tenant and then rejects a halted one
before any handler runs. A flag nothing reads halts nothing — without this read,
a halted tenant's still-valid API key would keep authenticating and keep
ingesting reviews.

Two properties of that check are deliberate:

- **It lives in the shared authentication dependency, not on individual
  routes.** Every tenant-authenticated surface — ingestion, provider endpoints,
  analytics, dead-letter queue operations, and the tier-aware rate budget that
  chains off it — inherits the gate from one place. A route added later cannot
  forget to apply it.
- **The rejection is an authorization failure, not an authentication one.** The
  credential is genuine and correctly presented; the tenant is deliberately
  refused service. Reporting it as an authentication failure would tell a halted
  tenant their key is invalid and send them into a credential-rotation loop that
  cannot fix anything.

The operator kill endpoint is gated by the separate operator-level key rather
than by tenant authentication, so halting a tenant never locks operators out of
inspecting or repeating the action.

### 6. Addressable graph state

The agent checkpointer key is a canonical, tenant-scoped composite identifier
produced by a single builder and validated by a matching parser. The parser
rejects malformed input — wrong segment count, wrong labels, non-UUID segments,
and oversized payloads (length-capped before parsing, which also defeats
path-traversal strings).

Embedding the tenant makes the state namespace tenant-addressable and lets a
forged key be rejected at the boundary rather than handed to the state store
verbatim. This is a boundary-validation invariant, not the isolation control:
cross-tenant *access* to stored state is prevented one layer up, where the
approval path performs a tenant-scoped lookup and 404s before it ever reads
graph state — because the checkpointer itself keys only on the identifier it is
given.

---

## 8. Defense layers around the pipeline

**Authentication.** Tenants authenticate with an API key; only its hash is
stored. Privileged operator routes use a **separate** operator-level key — a
single operator secret, not a tenant identity — compared in constant time, with
responses that encode no reason for the rejection so they can't be used as an
oracle. Provider-specific ingestion endpoints add a *second factor* (an HMAC
signature over the raw body, or a bearer token) proving the request genuinely
came from the named upstream and not merely from someone who learned the
tenant's API key. Chat-platform and billing webhooks verify platform signatures
with a replay window.

Every one of these **fails closed**: if the server-side secret is unset, the
route rejects all traffic rather than running open.

Replay protection is implemented only where the upstream scheme includes a
signed timestamp to anchor a freshness window to; where a provider offers no
such anchor, the absence is documented rather than faked.

**Rate limiting, two layers.** A blanket per-tenant sliding window, keyed on a
hash of the credential, runs on every keyed request — cheap, no database
round-trip, catching infrastructure-level abuse. Inside that, a tier-aware
budget reads the tenant's usage tier and applies to the model-expensive routes
only. Tier resolution **fails closed** to the most restrictive tier on a missing
or malformed value. This inner layer is the primary defense against model-credit
exhaustion, which is the attack that costs real money rather than just
availability. Both use in-process counters and are therefore single-replica;
moving to a shared store is a prerequisite for scaling out horizontally.

**Dead-letter queue.** Every failure path writes a dead-letter row. Retries
update the existing row rather than creating new ones, so each underlying review
has at most one open entry, and a successful retry closes it. Operators list and
replay through admin endpoints.

**Notification routing is state-driven**, and the routing itself is a security
control: a PII-flagged terminal goes to an operator-only security channel
carrying no review or draft text, while an ordinary flagged review goes to the
human-review channel with content that upstream stages already certified clean.
Notification is best-effort and runs outside any database transaction — a
messaging outage must never roll back an already-persisted review.

---

## 9. Observability

Structured JSON logging with four correlation keys bound for the lifetime of a
run: tenant, review, graph thread, and a request-scoped correlation id assigned
by middleware. Because background tasks outlive the request middleware's
context, these are re-bound inside the pipeline rather than relying on
inheritance — so an HTTP request log and the background work it triggered can be
stitched together across the async boundary.

Runs are bracketed by start/complete events carrying final status, wall-clock
duration, token counts, and flag counts. The guardrail emits its own
start/complete pair with an explicit pass/triggered decision and which checks
fired — the compliance audit trail. **Review text is never logged**; only
character counts.

Error tracking and log shipping are optional integrations that attach only when
configured, so local development and the test suite make no external calls and
ship no data off-box.

**Trace export is gated by a fail-closed latch.** Trace payloads include raw
node inputs and outputs — most critically the *unmasked* draft, since the
guardrail evaluates it pre-mask and hallucinated PII can appear there. That is
an unaudited third-party egress. Enabling tracing without a registered payload
scrubber **crashes startup** rather than silently shipping that content. The
guard also actively clears the underlying environment flag on every non-enabled
path, because the tracing library reads that variable directly and independently
of the application's settings model — a stray value arriving from the shell or
deploy config must not survive past the check.

---

## 10. Known limitations

Stated because a security document that lists only strengths is not useful:

- **Name redaction is heuristic** and trigger-based. It will miss names in
  ordinary prose. Full NER is the recall follow-up.
- **Rate limiting is in-process** and therefore single-replica. Horizontal
  scaling requires moving both layers to a shared store first.
- **Instructional-persistence detection is deterministic-only for the general
  case.** A semantic layer now supplements the rule-based check for two
  specific patterns — a draft echoing a customer's own PII back to them, and a
  draft disclosing details about the underlying software, client, or system —
  running after the deterministic sweep and failing closed: an error there
  blocks the draft rather than passing it, logged separately from an actual
  detection so an outage is never mistaken for a clean pass. Instructional-
  persistence patterns outside those two cases are still deterministic-only,
  by design, so a provider outage can't disable the baseline.
- **Provider replay protection is unimplemented** where the upstream scheme
  provides no signed timestamp to anchor to — see §8.
- **PII detection over-matches in one known case**, redacting more than
  necessary on certain identifier-shaped digit runs. Over-redaction, not
  under-redaction; tracked as non-blocking.
