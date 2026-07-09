# RFC-0006: The Transcript and State

- Status: Draft
- Ring: definition
- Depends on: RFC-0001

## Summary

Everything that happens in a run is an event on the **transcript**: one
canonical, append-only stream per run, covering every input, output, and
action taken by the harness, the model, and the definition. The transcript is
the storage abstraction. **State** is a set of typed views folded from it by
pure reducers; the conversation and the run's metrics are other views of the
same stream. History is append-only; only views rewind.

## Motivation

Agent systems accumulate state the way kitchens accumulate drawers: a
conversation log here, a scratchpad there, dispatcher bookkeeping, plan
objects mutated in place. Three costs follow: behavior cannot be explained
after the fact, recovery is unsafe because nobody knows what state existed at
any boundary, and transitions are untestable outside the full runtime. One
stream with pure folds answers all three.

## The stream

Every entry carries a common envelope: definition identity and version
(RFC-0013), adapter, entry type, source (main conversation or a named
sub-agent), per-source sequence number, UTC timestamp, and an opaque
adapter-specific detail payload with optional raw source data. The envelope
is the portable contract; harness-specific fidelity lives in the detail
payload, preserved but out of scope for cross-harness comparison.

The **entry-type vocabulary** is small and closed. Conversation types: user
message, assistant message, tool use, tool result, thinking, system event,
token usage, error. Definition-plane types: state event, policy decision,
feedback delivered, completion verdict. Plus **unknown**: anything a harness
emits that cannot be mapped is preserved raw under `unknown` — never dropped,
never mislabeled.

Ordering and reliability:

- Within one source, sequence numbers are strictly monotonic. Entries
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

Appending to the transcript is the only mutation path: dispatch means
append-then-fold. Convenience accessors MUST desugar to events, so every
change is a recorded entry — state is an explanation, not a cache.

**Working state versus history.** Every slice is either *state* (the agent's
working memory — snapshotted and restored by transactions, RFC-0003) or *log*
(a direct projection of history — untouched by rollback). Because the
substrate is append-only, the second rule is structural: history cannot be
erased, only views rewound. This resolves the apparent paradox of RFC-0003 —
"a failed tool leaves no trace" means no trace *in working state*; the
attempt is still on the transcript.

**Snapshots.** The library MUST support capturing an immutable snapshot of
the folded state and restoring from one, with serialization stable enough to
outlive the process — snapshots are the transaction mechanism, the
checkpoint/resume mechanism, and the "state after" of the run record. A
snapshot is a materialized checkpoint of the fold, not a second source of
truth.

**Storage independence.** The transcript and its views are protocols, not
data structures: in-memory for short runs, append-only files for crash
recovery and audit, remote backing for fleets. The requirements are
behavioral — identical semantics across backings. A definition MUST NOT need
to know how its stream is stored.

## In-process observation

Dispatch doubles as the telemetry spine: subscribers observe events
synchronously and in order, and subscriber failures are isolated — logged,
never propagated — because observability must never break the run.
Cross-process durability is explicitly not this mechanism's job; that is work
distribution (RFC-0010).

## Honesty about replay

An event stream invites the claim "fully replayable." Resist it: external
evidence expires, privacy boundaries forbid retention, models are
nondeterministic. The normative bar is **explainability** — the transcript
and run record MUST preserve intent, evidence references, decisions, and
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

## Compatibility surface

An adapter certification suite MUST assert:

- Envelope completeness and per-source sequence monotonicity for every entry,
  on every harness.
- Causal ordering (decision → execution → result) and lifecycle bracketing
  (start and stop entries around the run).
- The same logical scenario yields the same canonical entry-type sequence
  across harnesses — the transcript is the oracle for adapter equivalence
  (RFC-0012).
- Unmappable native events surface as `unknown` with raw data preserved.
- Snapshot → restore round-trips working state exactly and leaves history
  intact through transaction rollback.
