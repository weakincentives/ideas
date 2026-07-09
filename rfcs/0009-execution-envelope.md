# RFC-0009: The Execution Envelope — Deadlines, Budgets, Leases, and Liveness

- Status: Draft
- Ring: control plane
- Depends on: RFC-0003, RFC-0005, RFC-0008, RFC-0010

## Summary

Every unattended run executes inside an **envelope**: a wall-clock deadline, a
resource budget, and a liveness contract. The envelope is declared per run,
propagated to every layer (provider calls, tool calls, feedback, completion),
and enforced at defined checkpoints. Liveness is proven, not presumed:
workers emit **heartbeats tied to actual work**, message leases extend only
when heartbeats occur, and a **watchdog** kills processes whose heartbeats go
stale. The design principle throughout: a stuck agent must become someone
else's agent quickly, and a spending agent must hit a wall someone chose.

## Motivation

An attended agent that hangs gets Ctrl-C. An attended agent that loops gets
noticed by the human paying attention — and the human paying. Unattended,
both failure modes are silent and unbounded: a hung worker holds its work
item invisible forever; a looping agent converts model tokens into heat.
Meanwhile the infrastructure-level answer — hard timeouts around the whole
process — is too blunt: it cannot distinguish "stuck" from "slow but
progressing," and it loses everything the run accomplished.

The envelope gives the control plane graduated, legible bounds: soft
awareness (feedback), hard checkpointed limits (deadline/budget), work-proven
lease extension, and last-resort termination (watchdog) — each mechanism
covering the failure mode the previous one cannot.

## Design

### Deadlines

- A deadline is a **wall-clock instant** (timezone-aware), set by the caller
  per run, optionally defaulted by configuration.
- It propagates everywhere: into the rendered definition's metadata, provider
  calls, every tool context, feedback contexts, and completion contexts. One
  deadline governs the whole evaluation including tools — there are no
  per-layer private deadlines that add up to more than the caller allowed.
- Enforcement is **checkpointed**: before provider calls, before each tool
  execution, and during response finalization. Mid-flight cancellation of an
  in-progress model call or tool is explicitly not promised; what is promised
  is that no *new* work starts past the deadline and the overshoot is bounded
  by the longest single operation.
- Expiry surfaces as a typed, phase-attributed error (request / tool /
  response), so run records show *where* time ran out.

Deadlines interact with the definition layer deliberately: a deadline-aware
feedback provider (RFC-0005) warns the agent as time shortens, and completion
gates are bypassed-with-record on expiry — the envelope always wins over the
desire for more turns.

### Budgets

- A budget bounds **consumption**: input tokens, output tokens, total tokens,
  and optionally cost, alongside the deadline.
- One **tracker** per run accumulates usage across provider calls, retries,
  and continuation rounds — retries do not get fresh budgets, which is what
  actually caps runaway loops.
- Enforcement points mirror deadlines: after every provider response, after
  every tool call, at completion. Breach raises a typed error carrying the
  consumption record.
- The tracker is a capability (RFC-0008) visible to tools and feedback, so
  definitions can steer ("you have used 80% of budget") before the wall.

### Heartbeats: liveness as proof of work

A heartbeat is a monotonic-clock timestamp the worker refreshes when it does
something real: receiving work, completing a tool call, making progress
inside a long-running handler (via the tool context, RFC-0003). Adapters MUST
arrange for beats around native harness operations too, or long native
operations become indistinguishable from hangs.

The crucial design decision is what heartbeats are **not**: a background
thread beating on a timer. A timer proves the process is scheduled; it says
nothing about progress. Every consumer of heartbeats below depends on beats
meaning *work happened*.

### Lease extension tied to heartbeats

Work items come from queues with visibility timeouts (RFC-0010). Long runs
outlive reasonable timeouts, so leases must extend — but extension must not
be automatic:

- On each heartbeat, if at least the configured interval has passed since the
  last extension, extend the item's invisibility by the configured amount.
- No beats → no extensions → the lease expires and the item is redelivered to
  a healthy worker. **A stuck worker loses its work by default.** This is the
  property that makes unattended fleets self-healing, and it emerges from the
  heartbeat/extension coupling with no coordinator.
- Extension failures (expired handle, transient backend errors) are logged
  and non-fatal: extension is an optimization for continuity, never a
  correctness dependency — correctness comes from idempotent processing
  (RFC-0010).

### Watchdog and health

- A **watchdog** monitors heartbeat age; past a stall threshold it terminates
  the process — immediately and unconditionally, on the theory that a stuck
  worker cannot run its own graceful shutdown, and the orchestrator's restart
  is the recovery path.
- **Health probes** (liveness/readiness) expose the same signals to the
  orchestrator; readiness SHOULD incorporate heartbeat freshness so a
  stalling worker stops receiving traffic before the watchdog fires.
- The thresholds form an arithmetic system that MUST hold:
  `visibility timeout > watchdog threshold + max processing step`, and
  `watchdog threshold > poll wait + max processing step`, with the check
  interval a small fraction of the threshold. Implementations SHOULD validate
  these relationships at configuration time rather than letting operators
  discover them as redelivery storms.

### The layered picture

| Layer | Detects | Responds |
| --- | --- | --- |
| Feedback (0005) | Time/budget pressure | Advises the agent |
| Deadline/budget checkpoints | Limit reached | Typed, phase-attributed halt |
| Lease + heartbeat | Worker stopped progressing | Work item redelivered elsewhere |
| Watchdog | Process unrecoverable | Kill; orchestrator restarts |

## Anti-patterns

- **Timer-thread leases.** Extending leases on a schedule keeps dead workers'
  work invisible for hours — the exact failure the mechanism exists to
  prevent.
- **Per-layer timeout stacking.** Independent timeouts at adapter, tool, and
  queue layers that don't derive from the one deadline produce runs that die
  early or overrun mysteriously.
- **Fresh budgets on retry.** Letting each attempt spend the full budget
  turns a retry loop into an unbounded spend.
- **Graceful-shutdown faith.** Sending a stuck worker a polite signal and
  waiting; the watchdog exists because politeness already failed.

## Compatibility surface

An adapter certification suite MUST assert:

- An already-expired deadline fails fast with the correct phase before any
  provider call.
- A deadline expiring mid-run halts at the next checkpoint with the correct
  phase attribution; no tool starts after expiry.
- Budget breach halts at the defined checkpoints; the tracker's record
  matches provider-reported usage across retries.
- Tool-context beats reach the lease extender (observable via extension
  calls on a fake queue with a fake clock).
- Completion-gate bypass on envelope exhaustion is exercised and recorded.

## Open questions

- Should budgets extend beyond tokens/cost to **effect budgets** (N writes, N
  external calls) as a first-class envelope dimension, unifying with policy
  means-limits (RFC-0004)?
- Checkpointed enforcement bounds overshoot by the longest single operation;
  is a cooperative cancellation contract for tools (a token handlers must
  poll) worth its complexity?
- Envelope inheritance for sub-agents: fixed subdivision, shared tracker, or
  definition-declared split? Shared tracker is safest but couples sibling
  failure modes.
