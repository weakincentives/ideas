# RFC-0006: State as an Event Ledger

- Status: Draft
- Ring: definition
- Depends on: RFC-0001

## Summary

All run state flows through one mechanism: **typed events** dispatched to
**pure reducers** that produce new versions of **typed state slices**. There
is no hidden mutable state. Snapshots of the whole state are first-class,
serializable, and restorable — which is what makes tool transactions
(RFC-0003), policy state (RFC-0004), and honest run records (RFC-0011)
possible. The ledger distinguishes **working state** (rolled back with failed
transactions) from **append-only logs** (preserved through rollback), because
"what the agent believes" and "what actually happened" are different things
with different lifecycles.

## Motivation

Agent frameworks accumulate state the way kitchens accumulate drawers: a
transcript here, a scratchpad there, tool-call bookkeeping in the dispatcher,
plan objects mutated in place. Three costs follow:

- **Inexplicability.** After an unattended run misbehaves, the operator needs
  to know what the agent knew, when it knew it, and what constrained it.
  Reconstructing that from ad-hoc mutation requires archaeology.
- **Unsafe recovery.** Rollback, retry, and resume all require knowing
  exactly what state existed at a boundary. Mutable objects with aliases
  cannot answer that.
- **Untestability.** State transitions buried in runtime glue can only be
  tested through the whole runtime.

Event-sourced state with pure reducers answers all three at once, and its
cost — a little ceremony per state type — is precisely the kind of cost that
pays off in systems nobody watches.

## Design

### Events, reducers, slices

- **Events** are typed values describing something that happened: a tool was
  invoked, a plan step was added, a policy decided, feedback was delivered.
  Event types are part of the definition's contract.
- **Slices** are typed collections of state, keyed by their type. A
  definition declares the slice types it cares about; the library provides
  standard slices (tool invocations, feedback history, policy state,
  visibility overrides).
- **Reducers** are pure functions `(current slice view, event) → operation`,
  where the operation is one of a small closed set: append, extend, replace,
  clear. Reducers MUST NOT mutate in place, perform I/O, or read anything but
  their inputs. Multiple reducers may respond to one event, each targeting
  its own slice.

Dispatch is the only mutation path. Convenience accessors (seed, append,
clear) MUST desugar to events so that *every* change is a recorded event —
this is what makes the ledger an explanation rather than a cache.

### Reading state

Reads go through typed accessors over slices: latest value, all values,
filtered queries. Reducers receive **lazy views** so that append-only
reducers need not load the slice at all — the design decision that lets a
file-backed slice run with O(1) appends and makes long runs cheap.

### Working state versus logs

Every slice carries a policy: **state** or **log**.

- **State slices** are the agent's working memory. Transactions snapshot and
  restore them; a failed tool leaves them untouched (RFC-0003).
- **Log slices** are the historical record — tool invocations, feedback,
  decisions. Rollback MUST preserve them: the fact that a tool ran and failed
  is itself part of history, and erasing it would falsify the run record.

This distinction is the resolution of an apparent paradox in RFC-0003
("failed tools leave no trace" — no trace *in working state*; the attempt is
still on the log).

### Snapshots

The library MUST support capturing an immutable snapshot of all state slices
and restoring from one. Snapshots MUST be serializable with stable, versioned
schemas (typed payloads survive the round trip), because they outlive the
process: they are the transaction mechanism, the checkpoint/resume mechanism,
and the "state after" section of the run record.

Restore semantics respect slice policy: state slices are replaced; log slices
are preserved.

### Storage independence

Slices are a protocol, not a data structure. In-memory backing suits short
runs; append-only file backing (one record per event, typed for polymorphic
reload) suits crash recovery and audit; remote backing suits fleets. The
normative requirements are behavioral: identical operation semantics across
backings, immutable views, and atomic replace. A definition MUST NOT need to
know how its slices are stored.

### Session hierarchy

Nested work (sub-agents, spawned evaluations) gets child state containers
linked to a parent, so aggregation tooling can walk the tree bottom-up.
Child state MUST NOT silently merge into the parent; boundaries between
reasoning contexts are part of the record.

### Event bus semantics

State dispatch doubles as the in-process telemetry spine. Delivery is
synchronous and in-order per dispatcher; subscriber failures are isolated
(logged, never propagated to the dispatching tool) — observability must never
break the run. Cross-process durability is explicitly *not* this mechanism's
job; that is work distribution (RFC-0010).

### Honesty about replay

An event ledger invites the claim "fully replayable". Resist overclaiming:
external evidence expires, privacy boundaries forbid retention, and models
are nondeterministic. The normative bar is **explainability**: the ledger
plus run record MUST preserve intent, evidence references, decisions, and
committed effects well enough to answer *why* the agent acted — even when
bit-for-bit re-execution is impossible.

## Anti-patterns

- **The escape hatch.** A single "misc" dict on the side, mutated directly,
  quietly becomes the real state and voids every guarantee here.
- **Effectful reducers.** A reducer that writes a file or calls a service
  turns replay and rollback into lies.
- **Unbounded working state.** Ledger semantics for logs, but working state
  should use replace/upsert shapes; treating everything as append-only makes
  reads quadratic and context assembly noisy.
- **Snapshot as afterthought.** Bolting serialization onto in-memory objects
  late produces schemas that break on reload exactly when a crashed run needs
  them.

## Compatibility surface

An adapter certification suite MUST assert:

- Standard events (render, tool invocation, execution completion) are
  dispatched with the same types, payloads, and ordering on every harness.
- Snapshot → restore round-trips state slices exactly and preserves log
  slices through transaction rollback.
- Serialized snapshots reload into equal state across process boundaries.

## Open questions

- Should event schemas support declared **compaction** (fold N events into a
  summary event) for very long runs, and if so, how does compaction interact
  with the explainability bar?
- Is a standard cross-implementation serialization for snapshots worth
  specifying (making run records portable between libraries), or is
  per-implementation stability enough?
- Where is the line between ledger events and transcript entries (RFC-0011)?
  Current position: the transcript is the model-conversation view, the ledger
  is the state view, and they cross-reference by correlation id — but a
  single unified log with two projections is a defensible alternative.
