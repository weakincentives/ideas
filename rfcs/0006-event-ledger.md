# RFC-0006: The Event Ledger

- Status: Draft
- Ring: definition
- Depends on: RFC-0001

## Summary

Everything that happens in a run is a typed event on the **ledger**: one
append-only, high-granularity stream per run — conversation turns, tool
invocations, guardrail decisions, state transitions, operational signals.
The ledger is the storage abstraction. Everything else is a view over it:
**state** is typed slices folded by pure reducers; the **transcript** — what
the model saw and did — is a deterministic projection for review,
certification, and analysis; metrics are a third. Not every state transition
appears in the transcript; nothing escapes the ledger. History is
append-only; only views rewind.

## Motivation

Agent systems accumulate state the way kitchens accumulate drawers: a
conversation log here, a scratchpad there, dispatcher bookkeeping, plan
objects mutated in place. Three costs follow: behavior cannot be explained
after the fact, recovery is unsafe because nobody knows what state existed at
any boundary, and transitions are untestable outside the full runtime. One
high-granularity stream with pure folds answers all three — and one curated
view answers the different question most readers are actually asking.

## The ledger

Every event carries a common envelope: definition identity and version
(RFC-0013), adapter, event type, source (main conversation or a named
sub-agent), per-source sequence number, UTC timestamp, and an opaque
adapter-specific detail payload with optional raw source data. The envelope
is the portable contract; harness-specific fidelity lives in the detail
payload.

Event types fall into four classes:

| Class | Examples | In the transcript? |
| --- | --- | --- |
| Conversation | User/assistant message, tool use, tool result, thinking, token usage, error | Yes |
| Guardrail | Policy decision, feedback delivered, completion verdict | Yes — these crossed the model boundary |
| State | Typed slice transitions declared by the definition | No |
| Operational | Timings, envelope consumption, lifecycle signals | No |

Plus **unknown**: anything a harness emits that cannot be mapped is preserved
raw under `unknown` — never dropped, never mislabeled.

Ordering and reliability:

- Within one source, sequence numbers are strictly monotonic. Events
  describing one effect MUST appear in causal order regardless of source —
  decision before execution before result. Interleaving across unrelated
  sources carries no guarantee, and consumers MUST NOT depend on it.
- Definition-plane appends are **authoritative**: transactional with the
  effect they record; a failed append fails the operation. Harness-derived
  mirroring is **best-effort**: its failures degrade the record, never the
  run, and are themselves recorded as warnings.
- Sub-agents are sources, not noise: nested agents emit under distinct source
  labels within the same run, and their state stays in child containers
  linked to the parent — boundaries between reasoning contexts are part of
  the record.

## State as views

- **Events** are typed values describing what happened; their types are part
  of the definition's contract.
- **Slices** are typed collections of state, keyed by type. The library
  provides standard slices (tool invocations, feedback history, policy state,
  visibility overrides); definitions declare their own.
- **Reducers** are pure functions `(slice view, event) → operation`, the
  operation drawn from a small closed set: append, extend, replace, clear.
  Reducers MUST NOT mutate in place, perform I/O, or read beyond their
  inputs. Reducers receive lazy views, so append-only folds need not load
  existing state — long runs stay cheap regardless of backing.

Appending to the ledger is the only mutation path: dispatch means
append-then-fold. Convenience accessors MUST desugar to events, so every
change is a recorded entry — state is an explanation, not a cache.

**Working state versus history.** Every slice is either *state* (the agent's
working memory — snapshotted and restored by transactions, RFC-0003) or *log*
(a direct projection of history — untouched by rollback). Because the
substrate is append-only, the second rule is structural: history cannot be
erased, only views rewound. This resolves the apparent paradox of RFC-0003 —
"a failed tool leaves no trace" means no trace *in working state*; the
attempt is still on the ledger.

**Snapshots.** The library MUST support capturing an immutable snapshot of
the folded state and restoring from one, with serialization stable enough to
outlive the process — snapshots are the transaction mechanism, the
checkpoint/resume mechanism, and the "state after" of the run record. A
snapshot is a materialized checkpoint of the fold, not a second source of
truth.

## The transcript view

The transcript is the projection of **what the model saw and did**: the
conversation events plus the guardrail moments that crossed the model
boundary — a delivered feedback, a denial reason, a completion verdict. State
transitions and operational bookkeeping stay ledger-only.

The membership rule is the boundary itself: an event belongs in the
transcript if and only if it was shown to the model or produced by it. The
projection MUST be deterministic — a pure filter over the ledger — so the two
can never disagree; regenerating the transcript from the ledger yields the
same view every time.

The transcript exists because most audiences need the story, not the
bookkeeping. It is the record a human reviews, the trace an analysis agent
walks first (RFC-0011), and — because harnesses legitimately differ in
internal granularity while the model-visible story must not differ — it is
the surface on which cross-harness equivalence is asserted: the
compatibility suite's oracle (RFC-0012).

## In-process observation

Dispatch doubles as the telemetry spine: subscribers observe events
synchronously and in order, and subscriber failures are isolated — logged,
never propagated — because observability must never break the run.
Cross-process durability is explicitly not this mechanism's job; that is work
distribution (RFC-0010).

## Honesty about replay

An event stream invites the claim "fully replayable." Resist it: external
evidence expires, privacy boundaries forbid retention, models are
nondeterministic. The normative bar is **explainability** — the ledger and
run record MUST preserve intent, evidence references, decisions, and
committed effects well enough to answer *why* the agent acted, even when
bit-for-bit re-execution is impossible.

## Anti-patterns

- **The escape hatch.** One "misc" dict mutated on the side quietly becomes
  the real state and voids every guarantee here.
- **Effectful reducers.** A reducer that writes a file turns replay and
  rollback into lies.
- **Everything append-only.** Ledger semantics for history, but working state
  wants replace/upsert shapes; append-everywhere makes reads quadratic and
  context assembly noisy.
- **Transcript stuffing.** Routing bookkeeping into the transcript because a
  consumer found it convenient. The transcript is the story; the ledger is
  the record. A consumer that needs more than the story queries the ledger.

## Compatibility surface

An adapter certification suite MUST assert:

- Envelope completeness and per-source sequence monotonicity for every
  ledger event, on every harness.
- Causal ordering (decision → execution → result) and lifecycle bracketing
  (start and stop events around the run).
- The same logical scenario yields the same transcript-view sequence across
  harnesses — the transcript is the oracle for adapter equivalence
  (RFC-0012). Full-ledger granularity MAY differ across harnesses; the
  model-visible story may not.
- The transcript regenerated from the ledger equals the transcript view
  produced during the run.
- Unmappable native events surface as `unknown` with raw data preserved.
- Snapshot → restore round-trips working state exactly and leaves history
  intact through transaction rollback.
