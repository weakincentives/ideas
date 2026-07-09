# RFC-0010: Work Distribution — Queues, Dead Letters, and Shutdown

- Status: Draft
- Ring: control plane
- Depends on: RFC-0006, RFC-0009, RFC-0011

## Summary

Unattended agents are workers in a distributed system, and the library must
say so out loud. Work arrives through a **typed queue protocol** with
visibility timeouts, explicit acknowledgment, and at-least-once delivery.
Failures that survive retry go to a **dead-letter queue** as forensic
envelopes, never into a log-and-drop. Workers shut down **cooperatively** —
finishing or returning in-flight work — and expose their liveness to
orchestrators. None of this is harness territory: how work reaches agents and
what happens when it fails is a control-plane contract the definition's owner
must be able to rely on across harnesses.

## Motivation

The gap between an agent demo and an agent fleet is almost entirely here. A
demo calls `run(request)`. A fleet needs: what happens when the process dies
mid-run? When the same request is delivered twice? When one poisonous request
crashes every worker that touches it? When the deploy system sends SIGTERM
during a forty-minute run? Every one of these has a standard answer from
queue-based systems; the work is specifying how those answers compose with
agent-specific realities — long, nondeterministic runs with model spend
attached (RFC-0009) and rich run records (RFC-0011).

## Design

### The queue protocol

A minimal, backend-agnostic contract, deliberately shaped like the
battle-tested cloud queue semantics rather than a novel invention:

- **Typed messages.** A queue carries one request type and knows its reply
  type; serialization failures are send-time errors, not worker-crash-time
  surprises.
- **Visibility timeout.** Receiving hides a message for a bounded period;
  processing must complete, extend (RFC-0009 leases), or the message
  reappears for another worker.
- **Explicit finalization.** `acknowledge` deletes; `nack` returns with an
  optional delay. Finalization is once-only; late operations on an expired
  receipt fail with a distinct error so races are visible.
- **Delivery count** travels with the message — the input to backoff and
  dead-letter decisions.
- **Reply routing.** A message may carry a reply-to reference; workers answer
  via the message, not via out-of-band channels. In distributed backings,
  reply references are names resolved through a **resolver** (registry plus
  factory), which is what makes per-request ephemeral reply queues and
  multi-tenant naming schemes possible without the worker knowing either.

**At-least-once is the contract.** Exactly-once is not on offer; consumers
MUST be idempotent. This composes with RFC-0003's transactions (clean state
on replay) and is the reason RFC-0009's send-result-before-acknowledge
ordering matters: the failure mode is duplicate work, never lost work.

### Backoff and retry

Redelivery backoff derives from delivery count (bounded exponential). The
run-level envelope is unchanged on retry — the budget tracker does not reset
(RFC-0009). Retries are for transient infrastructure and crash recovery;
*semantic* failure (the agent ran and produced a bad answer) is not a retry
matter but an evaluation matter (RFC-0014).

### Dead letters

A **dead-letter policy** decides when to stop retrying:

- After a configured maximum delivery count, or
- **Immediately** for error classes declared non-retriable (validation
  failures, authentication, content-policy violations), with an exclusion
  list for classes that must never dead-letter (rate limits, timeouts).
- Ambiguity defaults to retry — dead-lettering is the fail-closed direction
  for *work* only when repetition is provably useless.

Ownership is split by kind of knowledge. The **semantic classification** —
which error classes are permanent *for this agent* — belongs to the
definition, declared against the library's typed error taxonomy and carried
with the definition wherever it runs. The **mechanics** — destination queue,
maximum delivery count, retention — belong to deployment configuration. The
split only works because adapters are required to normalize every harness
failure into the typed taxonomy (RFC-0012): that error-translation layer is
the encapsulation boundary that makes a definition's failure semantics
portable across harnesses.

The dead letter itself is a forensic envelope, not a bare payload: original
message and id, source queue, delivery count, last error (type and message),
first-received and dead-lettered timestamps, and correlation ids linking to
run records (RFC-0011). Requirements:

- Dead-lettering MUST acknowledge the original (the queue unblocks).
- DLQ consumers MUST NOT dead-letter their own failures (no cascades);
  their failure mode is nack-with-long-delay and alerting.
- **Replay** — re-sending the original body to the source queue after a fix —
  SHOULD be supported as a first-class, idempotency-respecting operation.
- Retention on the DLQ SHOULD exceed the source queue's by an order of
  magnitude; dead letters are evidence.

### Graceful shutdown

Workers run a cooperative lifecycle:

- A **shutdown signal** (orchestrator SIGTERM, operator interrupt) sets a
  flag; the polling loop checks it between messages and stops receiving.
- In-flight work runs to completion within a **shutdown timeout**; received
  but unstarted messages are nacked immediately for prompt redelivery
  elsewhere.
- The recovery matrix must hold with no lost work: completed-and-acknowledged
  work stays done; interrupted work reappears after its visibility timeout;
  unstarted work reappears immediately.
- Multiple loops in one process (agent workers, evaluation workers, DLQ
  consumers) coordinate through one **loop group** that owns signal handling,
  health probes, and the watchdog (RFC-0009) — one process, one lifecycle
  authority.

### Verification note

Queue semantics are concurrency semantics, and their bugs (lost wakeups,
double delivery beyond contract, finalization races) do not reproduce in
example-based tests. Implementations SHOULD maintain machine-checked models
(state-machine specifications with safety invariants — TLA+-class tooling) of
the queue, lease, and shutdown state machines, and property-based tests
deriving from the same invariants. This is unusual rigor for an "agent
library" — and exactly the point of calling it a control plane.

## Anti-patterns

- **Acknowledge-then-process.** Inverts the loss guarantee; a crash after
  acknowledge loses work silently.
- **Log-and-drop failure handling.** A failed request that vanishes into a
  log line is invisible to remediation; the DLQ is the failure inbox.
- **In-band replies on a side channel.** Workers writing results to a shared
  store keyed by convention reinvent reply routing without its guarantees.
- **Queue as event bus.** Broadcasting telemetry through the work queue (or
  distributing work through the in-process event dispatcher, RFC-0006)
  conflates two tools with opposite delivery semantics.

## Compatibility surface

This RFC binds the library and its backends more than harness adapters, but
the suite MUST assert across queue backends:

- Visibility, finalization, delivery-count, and reply semantics behave
  identically over every backing (in-memory, networked) via a shared
  validation suite — including expired-receipt and double-finalization
  errors.
- The dead-letter flow: max-delivery, immediate classes, excluded classes,
  envelope completeness, source acknowledgment.
- The shutdown recovery matrix under induced interruption at each phase.

## Open questions

- Message priority and fairness (starvation of long jobs by short ones) —
  first-class queue feature, or deployment topology (separate queues per
  class)?
- Should the reply pattern standardize *progress* replies (multiple
  non-final replies before finalization) as an alternative to out-of-band
  status stores for long runs?
