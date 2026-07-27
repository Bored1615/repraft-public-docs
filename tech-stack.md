# Repraft — Tech Stack

What the system is built on, and why each choice was made. Versions are pinned
by a committed lockfile in the application repository; this document describes
the shape of the stack rather than exact pins.

---

## At a glance

| Layer | Choice |
|---|---|
| Language | Python 3.12+ |
| Package / environment manager | `uv` (lockfile-pinned, reproducible) |
| API framework | FastAPI, fully async |
| ASGI server | Uvicorn |
| Agent orchestration | LangGraph + LangChain Core |
| Model provider | Anthropic Claude, via the LangChain Anthropic integration |
| Agent state | LangGraph PostgreSQL checkpointer |
| Database | PostgreSQL |
| ORM | SQLAlchemy 2.x, async, typed mappings |
| Drivers | An async driver for the ORM; a separate driver for the checkpointer |
| Migrations | Alembic |
| Validation & settings | Pydantic v2 + pydantic-settings |
| Retry / backoff | Tenacity |
| Logging | structlog |
| Error tracking | Sentry (optional) |
| Log aggregation | External log platform (optional) |
| Tracing | LangSmith (optional, gated) |
| Rate limiting | In-process sliding windows + SlowAPI |
| Token signing | itsdangerous |
| Testing | pytest, pytest-asyncio, pytest-cov, mutmut |
| Packaging | Multi-stage Docker image |
| Hosting | Container platform with managed PostgreSQL |

---

## Runtime and API

**Python 3.12+, FastAPI, async throughout.** The workload is almost entirely
I/O-bound — database round-trips, model calls, outbound notifications — which is
the case async concurrency is actually good at. FastAPI brings Pydantic-based
request validation as a first-class part of the framework rather than a
bolted-on layer, which matters here because **validation is a security control**:
sanitation and hostile-input detection live in the schema layer and run before a
handler body executes.

The single most consequential API decision is that **ingestion returns `202`
before any model work starts**. The expensive, failure-prone work runs as a
background task. An aggregator gets a fast acknowledgement instead of a request
held open across a multi-second model call, so a slow model can't cause an
upstream timeout and the retry storm that follows.

**`uv`** manages dependencies and the environment. The committed lockfile is
what makes the container build reproducible and the same resolution run in CI,
locally, and in production.

---

## Agent layer

**LangGraph** rather than a linear chain or a free-roaming agent loop. The
reason is control flow: this pipeline is a state machine with a conditional
refinement loop, several distinct terminal states, and hard requirements about
which transitions are reachable. LangGraph makes those edges explicit and
inspectable — you can read the routing and know that no path leads from a
guardrail violation to approval. A free-roaming agent that decides its own next
step cannot offer that property, and the property is the product.

Deterministic entry and exit stages wrap the model calls. The sanitation node
makes **no model call at all**, so an attacker cannot pivot the sanitizer
itself, and the guardrail is rule-based so it keeps working precisely when the
model is misbehaving.

**Claude** is the drafting and classification model, reached through the
LangChain Anthropic integration. Structured output is used for classification so
the sentiment label is a validated value rather than parsed prose.

**Tenacity** owns retry policy for model calls in one place: bounded attempts
with exponential backoff, retrying only transient connection, timeout,
rate-limit, and server errors. Authentication and bad-request failures are not
retried — they aren't network faults, and retrying only delays the inevitable.
Exhausted retries raise a typed error that the graph converts into a retryable
terminal state rather than a crash.

An **offline mock model** stands in for the real client behind a configuration
flag, exposing the same two surfaces the graph uses. This lets load and
endurance tests exercise the entire pipeline — including the guardrail, the
database, and the checkpointer — with zero network calls and zero token spend.

---

## Data layer

**PostgreSQL** with **async SQLAlchemy 2.x**, typed declarative mappings, and
**Alembic** migrations applied at container start.

Project rule: **no raw SQL** — all access goes through ORM constructs. This is
partly injection hygiene, but the larger reason is that it lets the tenant
filter be enforced *structurally*, by a repository layer that binds the
predicate into every statement it builds. Raw SQL scattered across handlers
would make that impossible to guarantee.

**Two connection pools** open at startup against the same database. This is not
redundancy: the agent framework's PostgreSQL checkpointer requires a different
driver than the async ORM engine uses, and they cannot share. The ORM pool runs
with pre-ping so a connection killed by a database restart is recycled rather
than handed to a request; the checkpointer pool is configured to the settings
its saver requires, with a checkout timeout sized to absorb webhook bursts.

---

## Configuration and secrets

**pydantic-settings** is the single validated entry point for configuration —
project rule, no ad-hoc environment reads. Secret-typed fields keep credentials
from leaking into logs and reprs by accident.

Configuration is a **startup gate, not just a data structure**. Cross-field
validators refuse to boot the process on unsafe combinations — for example, a
non-local environment still carrying the well-known development signing secret,
or trace export enabled without a registered payload scrubber. A bad
configuration crashes at startup, where it's obvious, rather than at the first
request, where it isn't.

Optional integrations are genuinely optional: when their configuration is
absent, no client is initialized and no network handler is attached, so local
development and the test suite ship nothing off-box.

Approval links are signed with **itsdangerous** timed serializers — expiring,
scoped tokens rather than guessable identifiers in a URL.

---

## Observability

**structlog** emitting JSON in deployed environments and human-readable console
output locally. Correlation keys are bound via context variables so they
propagate automatically to every log call in a run, including inside graph
nodes, without threading a logger through every signature.

The processor chain is itself a control: a PII-masking processor sits after
context binding and before rendering, so every line — application and foreign
library alike — is masked from one place before it reaches stdout or an external
aggregator.

**Sentry** for error tracking and **an external log platform** for aggregation,
both attaching only when configured. **LangSmith** tracing is available but
**gated shut by default**: trace payloads carry raw node inputs and outputs
including the unmasked draft, so enabling it without a registered payload
scrubber crashes startup rather than opening an unaudited third-party PII
egress.

---

## Testing

**pytest** with **pytest-asyncio**, plus **pytest-cov** for coverage and
**mutmut** for mutation testing — the latter because coverage tells you a line
executed, not that a test would fail if the line were wrong, and for
security-critical branches that difference matters.

The default suite is **hermetic**: the model is stubbed and the database is
in-memory, so it needs no network and spends no tokens. Beyond unit and
integration tests there is an adversarial layer — prompt-injection and PII
corpora, tenant-isolation probes, authentication and rate-limit checks, and
ordering guarantees — that pins the architectural invariants so a regression
fails the build. Database-level checks that need real PostgreSQL semantics are
opt-in and skip cleanly when no live instance is configured.

Static analysis and dependency scanning run over the codebase for lint,
security patterns, dependency advisories, and committed-secret detection.

---

## Packaging and deployment

A **multi-stage Docker image**: a builder stage resolves and installs
dependencies from the lockfile with build-cache mounts, and a slim runtime stage
copies in only the resulting virtual environment and application code. The
runtime **runs as a non-root user**, and build toolchains never reach the final
image.

Deployment targets a container platform with managed PostgreSQL. The start
command applies migrations and then execs the ASGI server, so one image and one
process constitute a deploy, with a health endpoint for the platform's checks
and a restart policy on failure.

One notable detail: managed PostgreSQL add-ons commonly inject a connection URL
with a scheme that resolves to a **synchronous** driver, which isn't installed
in an all-async stack. Normalizing that scheme happens inside the settings model
rather than in a shell wrapper, so *every* entrypoint gets the fix — including
one-off management commands run against the deployed environment, not just the
server boot path.

---

## Deliberate non-choices

- **No LLM-based guardrail as the primary control.** A model judging model
  output fails exactly when it's needed most. An LLM judge is planned as a
  *second* opinion layered on top of the deterministic baseline, never as a
  replacement for it.
- **No vendor SDK for chat notifications.** The outbound message format is
  modelled as a narrow set of typed schemas covering only the blocks actually
  emitted, with per-field length limits enforced at construction. Errors surface
  as validation failures at build time rather than as a rejected API call
  visible only in a log line. A full SDK would be a large dependency for a small
  surface.
- **No shared cache/queue yet.** Rate limiting and background work run
  in-process. This is a correct fit for a single replica and an explicit,
  documented ceiling on horizontal scaling rather than an oversight — adding
  distributed infrastructure before it's needed buys operational burden, not
  reliability.
- **No dashboard.** The product is an integration; the entire surface is JSON
  over HTTP, and the human-in-the-loop step runs through existing operator
  channels.
