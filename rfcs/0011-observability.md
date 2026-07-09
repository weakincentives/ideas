# RFC-0011: The Run Record — Transcripts, Bundles, and Queryability

- Status: Draft
- Ring: control plane
- Depends on: RFC-0006, RFC-0012

## Summary

Every run must be able to explain itself after the fact, to someone who
wasn't there, without the process that produced it. Three contracts deliver
that: the **transcript** — the run's single storage abstraction: one
canonical, append-only event stream of every input, output, and action taken
by the harness, model, and definition, regardless of which harness produced
it, consumed through **views** shaped for different audiences; the **run
record** (debug bundle) — a self-contained, integrity-checked archive of the
transcript and everything else about one run; and **queryability** — the run
record must answer structured questions with standard query tooling, because
its most important reader is another program (increasingly, another agent).

## Motivation

Unattended operation moves debugging from "watch it happen" to "reconstruct
what happened." Harnesses each have native logs, in their own formats, with
their own gaps; a fleet running one definition across three harnesses
otherwise needs three forensic skill sets and produces three incomparable
records. And the sheer volume of runs means human reading is the exception:
the run record's design target is programmatic analysis — dashboards,
regression diffing, and analysis agents (RFC-0014) — with humans dropping in
at the anomalies.

Observability is also the portability keystone: RFC-0012's compatibility
suite uses the transcript as its oracle. If two harnesses produce structurally
equivalent transcripts for the same definition, the adapter preserved
semantics; if they can't be compared, nothing else about portability can be
proven either.

## Design

### The transcript: one substrate, many views

One schema, many sources, one stream per run. Every entry carries a common
envelope:

| Field | Meaning |
| --- | --- |
| definition identity | Which definition (and version, RFC-0013) produced this |
| adapter | Which adapter/harness ran it |
| entry type | One of a small canonical vocabulary (below) |
| sequence number | Strictly increasing, per source, no gaps |
| source | Main conversation or a named sub-agent |
| timestamp | UTC, capture time |
| detail | Adapter-specific payload, opaque to generic consumers |
| raw | Optional original source data |

The canonical **entry-type vocabulary** is deliberately small. Conversation
types: user message, assistant message, tool use, tool result, thinking,
system event, token usage, error. Definition-plane types: state event, policy
decision, feedback delivered, completion verdict. Plus **unknown** for
forward compatibility. Adapters map their harness's native stream (file
tails, protocol notifications, in-memory events) into this vocabulary, and
the library appends the definition-plane entries itself; anything unmappable
becomes `unknown` with its raw form preserved, never dropped and never
mislabeled.

Normative semantics:

1. **Two reliability classes.** Definition-plane entries (state events,
   policy decisions) are authoritative — their append is transactional with
   the effect they record, and a failed append fails the operation.
   Harness-derived entries (conversation mirroring) are best-effort — their
   failures never fail the run; they degrade the record and are themselves
   recorded as warnings.
2. **Ordering.** Within one source, sequence numbers are strictly monotonic;
   causal pairs (tool use before its tool result) MUST be ordered.
3. **Fidelity split.** The envelope is the portable contract; harness detail
   lives in `detail`/`raw`, preserved but out of scope for cross-harness
   comparison. This split is what lets the schema be both stable and honest.
4. **Sub-agents are sources**, not noise: a harness that spawns nested agents
   emits their transcripts under distinct source labels within the same run.

The transcript is the substrate, not one view among several. The
**conversation view** (the model dialogue), the **state view** (typed slices
folded by reducers, RFC-0006), and the **operational view** (timings, token
usage, envelope consumption) are all projections of the same stream, so they
cannot disagree with each other — a tool-use entry, its policy decision, its
state effect, and its metrics are the same events read through different
lenses, joined by correlation ids. New audiences get new views, not new
logs.

### The run record

A self-contained archive per run, with a deterministic layout, containing at
minimum: the request and response; state snapshots before and after; the full
structured log including the transcript; configuration (definition identity
and version, adapter, envelope); correlation ids; metrics (timing phases,
token consumption, budget state); the error if any; and — within declared
size budgets — the workspace contents.

For reproducibility, the record SHOULD also capture the execution
environment: platform, runtime versions, dependency manifest, relevant
(redacted) environment variables, and source-control position including
uncommitted changes. "It only fails in production" is answerable only if
production described itself.

Normative properties:

- **Self-contained.** Diagnosis needs the record and standard tooling —
  no access to the original process, host, or dashboards.
- **Atomic.** The record exists completely or not at all; a crash during
  capture must not leave plausible-looking partial records.
- **Integrity-checked.** A manifest lists contents with checksums and format
  version.
- **Graceful degradation.** Capture failures (oversized file, unreadable
  path) are recorded as omissions, never as run failures.
- **Redaction-aware.** Secret material never enters the record (RFC-0007);
  environment capture is filtered by default.
- **Lifecycle-managed.** Retention policies (count, age, total size) and
  export handlers to external storage are part of the contract, because a
  fleet produces records faster than any disk grows.

### Correlation

Three identities, all in every record and every log line: the **request id**
(stable across redeliveries of the same work item), the **run id** (fresh per
execution attempt), and the **attempt number** — plus pass-through slots for
distributed-tracing ids so agent runs join wider system traces. The
request/run distinction is what makes at-least-once delivery (RFC-0010)
auditable: one request, N runs, each with its own record.

### Queryability

A run record MUST be queryable with a standard structured query language
without custom parsers: schema discovery, then queries over logs, transcript,
tool calls, errors, state slices (each typed slice queryable as its own
relation), configuration, and metrics. Two design consequences:

- Typed events and slices (RFC-0006) are what make this cheap — schema falls
  out of the types.
- The query surface is the natural interface for **analysis agents**: an
  agent handed a run record and a query tool can investigate failures at
  fleet scale (RFC-0014). Designing the record for that consumer — stable
  schemas, discoverability, no tribal knowledge required — is a hard
  requirement, not polish.

## Anti-patterns

- **Logs as the record.** A pile of interleaved log lines without envelope,
  ordering, and correlation contracts is data, not explanation.
- **Blocking telemetry.** Any path where observability failure fails the run
  inverts priorities.
- **Harness-native lock-in.** Treating one harness's transcript format as
  the system of record makes every other harness a second-class citizen and
  the compatibility suite unwritable.
- **Unbounded capture.** Recording everything with no size budgets or
  retention turns the forensic system into the outage.

## Compatibility surface

An adapter certification suite MUST assert:

- Envelope completeness and sequence monotonicity for every entry, on every
  harness.
- Causal ordering (tool use before tool result) and lifecycle bracketing
  (start/stop entries around the run).
- The same logical scenario yields the same canonical entry-type sequence
  across harnesses (the transcript-as-oracle property).
- Unmappable native events surface as `unknown` with raw preserved.
- Run records verify against their manifests, and the transcript extracted
  from a record equals the transcript emitted during the run.

## Open questions

- How far should cross-harness *content* normalization go — e.g., token-usage
  entries have wildly different native granularity; is per-run aggregate
  parity enough?
- Should run records support **streaming export** (record grows during the
  run, finalized at the end) for very long runs where post-hoc capture risks
  losing the tail?
- Privacy tiers: a standard way to declare fields/artifacts as
  retention-limited within a record, so one archive can serve both debugging
  and compliance clocks?
