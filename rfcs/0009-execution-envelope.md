# RFC-0009: The Execution Envelope

- Status: Draft
- Ring: control plane
- Depends on: RFC-0003, RFC-0005, RFC-0008, RFC-0010

## Summary

Every unattended run executes inside an **envelope**: a wall-clock deadline,
a resource budget, and a liveness contract. The envelope is declared per run,
propagated to every layer, and enforced at defined checkpoints. Liveness is
proven, not presumed: heartbeats fire on actual work, message leases extend
only when heartbeats occur, and a watchdog kills processes whose heartbeats
go stale. The principle throughout: a stuck agent must become someone else's
agent quickly, and a spending agent must hit a wall someone chose.

## Motivation

Attended, a hung agent gets Ctrl-C and a looping agent gets noticed.
Unattended, both are silent and unbounded: a hung worker holds its work item
invisible forever; a looping agent converts tokens into heat. Whole-process
timeouts are too blunt — they cannot distinguish stuck from slow, and they
lose everything the run accomplished. The envelope provides graduated,
legible bounds instead.

## Deadlines

A deadline is a wall-clock instant, set per run, propagated everywhere — into
rendered-definition metadata, provider calls, every tool context, feedback
and completion contexts. One deadline governs the whole evaluation; there are
no per-layer private deadlines that sum to more than the caller allowed.

Enforcement is **checkpointed**: before provider calls, before each tool,
during response finalization. Mid-flight cancellation is not promised; what
is promised is that no new work starts past the deadline, bounding overshoot
by the longest single operation. Expiry surfaces as a typed, phase-attributed
error, so run records show *where* time ran out. The definition layer
cooperates deliberately: a deadline-aware feedback provider (RFC-0005) warns
the agent as time shortens, and completion gates are bypassed-with-record on
expiry — the envelope always wins over the desire for more turns.

## Budgets

A budget bounds consumption — input, output, and total tokens, optionally
cost — tracked by one accumulator per run across provider calls, retries, and
continuation rounds. Retries do not get fresh budgets; that is what actually
caps runaway loops. Enforcement mirrors deadlines: after every provider
response, after every tool call, at completion, with breach raising a typed
error carrying the consumption record.

The tracker is a capability (RFC-0008) visible to tools and feedback, so
definitions can steer before the wall. And sub-agent spend is parent spend:
work a run spawns consumes the spawning run's envelope. How a definition
apportions among children is its choice; that no spend escapes attribution is
not.

## Liveness: heartbeats, leases, watchdog

A **heartbeat** is a monotonic timestamp refreshed when the worker does
something real: receiving work, completing a tool call, progressing inside a
long handler (via the tool context, RFC-0003). Adapters arrange beats around
native harness operations too, or long native operations become
indistinguishable from hangs. What heartbeats are *not*: a background thread
on a timer — that proves the process is scheduled, not that work is
happening.

**Lease extension is tied to heartbeats.** Work items carry visibility
timeouts (RFC-0010); long runs must extend them — but only on proof of work.
On each beat, if the configured interval has passed, extend the lease. No
beats, no extensions: the lease expires and the item is redelivered to a
healthy worker. **A stuck worker loses its work by default** — the property
that makes fleets self-healing with no coordinator. Extension failures are
logged and non-fatal; extension is an optimization for continuity, never a
correctness dependency — correctness comes from idempotent processing
(RFC-0010).

The **watchdog** monitors heartbeat age and, past a stall threshold, kills
the process — immediately, on the theory that a stuck worker cannot run its
own graceful shutdown; the orchestrator's restart is the recovery path.
Health probes expose the same signals, and readiness SHOULD incorporate
heartbeat freshness so a stalling worker stops receiving traffic before the
watchdog fires. The thresholds form an arithmetic that MUST hold — visibility
timeout exceeds watchdog threshold plus the longest processing step, which
exceeds the poll wait plus the same — and implementations SHOULD validate the
relationships at configuration time rather than letting operators discover
them as redelivery storms.

## The layered picture

| Layer | Detects | Responds |
| --- | --- | --- |
| Feedback (0005) | Time/budget pressure | Advises the agent |
| Deadline/budget checkpoints | Limit reached | Typed, phase-attributed halt |
| Lease + heartbeat | Progress stopped | Work redelivered elsewhere |
| Watchdog | Process unrecoverable | Kill; orchestrator restarts |

Each mechanism covers the failure the previous one cannot.

## Anti-patterns

- **Timer-thread leases.** Extending on a schedule keeps dead workers' work
  invisible for hours — the exact failure the mechanism exists to prevent.
- **Timeout stacking.** Independent per-layer timeouts that don't derive from
  the one deadline produce runs that die early or overrun mysteriously.
- **Fresh budgets on retry.** Each attempt spending the full budget turns a
  retry loop into unbounded spend.
- **Graceful-shutdown faith.** Politely signaling a stuck worker and waiting;
  the watchdog exists because politeness already failed.

## Compatibility surface

An adapter certification suite MUST assert:

- An already-expired deadline fails fast, before any provider call, with the
  correct phase.
- Mid-run expiry halts at the next checkpoint with correct phase attribution;
  no tool starts after expiry.
- Budget breach halts at the defined checkpoints; the tracker matches
  provider-reported usage across retries.
- Tool-context beats reach the lease extender (observable against a fake
  queue and fake clock).
- Completion-gate bypass on exhaustion happens and is recorded.
