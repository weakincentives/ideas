# RFC-0010: Work Distribution

- Status: Draft
- Ring: control plane
- Depends on: RFC-0006, RFC-0009, RFC-0011

## Summary

Unattended agents are workers in a distributed system, and the library must
say so out loud. Work arrives through a typed queue with visibility timeouts,
explicit acknowledgment, and **at-least-once delivery**. Failures that
survive retry become dead letters — forensic envelopes, never log lines.
Workers shut down cooperatively, finishing or returning in-flight work. How
work reaches agents and what happens when it fails is a control-plane
contract, not harness territory.

## Motivation

The gap between an agent demo and an agent fleet is almost entirely here. A
demo calls `run(request)`. A fleet must answer: what happens when the process
dies mid-run? When the same request arrives twice? When one poisonous request
crashes every worker that touches it? When the deploy system signals during a
forty-minute run? Queue-based systems answered these long ago; the work is
composing those answers with agent realities — long, nondeterministic runs
with model spend attached.

## The queue

A minimal, backend-agnostic contract shaped like the battle-tested cloud
queue semantics:

- **Typed messages.** A queue carries one request type and knows its reply
  type; serialization failures are send-time errors.
- **Visibility timeout.** Receiving hides a message for a bounded period;
  processing completes, extends the lease (RFC-0009), or the message
  reappears for another worker.
- **Explicit finalization.** Acknowledge deletes; negative-acknowledge
  returns with optional delay. Finalization is once-only, and late operations
  on an expired receipt fail with a distinct error so races are visible.
- **Delivery count** travels with the message — the input to backoff and
  dead-letter decisions.
- **Reply routing.** A message may carry a reply-to reference; workers answer
  via the message, and multiple replies before finalization are permitted —
  progress updates are ordinary replies. In distributed backings, reply
  references are names resolved through a registry-plus-factory, enabling
  per-request ephemeral reply queues without the worker knowing.

**At-least-once is the contract.** Exactly-once is not on offer; consumers
MUST be idempotent. This composes with RFC-0003's transactions (clean state
on replay) and fixes the ordering rule: send the result *before*
acknowledging, so the failure mode is duplicate work, never lost work.
Redelivery backoff derives from delivery count; the run's budget does not
reset on retry (RFC-0009). Retries are for transient infrastructure and crash
recovery — *semantic* failure (the agent ran and produced a bad answer) is
evaluation's problem (RFC-0014), not the queue's.

## Dead letters

A dead-letter policy decides when repetition is provably useless: after a
maximum delivery count, or immediately for error classes declared
non-retriable, with an exclusion list for classes that must never
dead-letter. Ambiguity defaults to retry.

Ownership is split by kind of knowledge. The **semantic classification** —
which error classes are permanent *for this agent* — belongs to the
definition, declared against the library's typed error taxonomy and carried
wherever the definition runs. The **mechanics** — destination, delivery
counts, retention — belong to deployment. The split works only because
adapters normalize every harness failure into the typed taxonomy (RFC-0012):
that translation layer is the encapsulation boundary that makes failure
semantics portable.

The dead letter itself is a forensic envelope: original message and id,
source queue, delivery count, last error type and message, timestamps, and
correlation ids linking to run records. Dead-lettering acknowledges the
original so the queue unblocks; dead-letter consumers never dead-letter their
own failures (no cascades); replay to the source after a fix is a
first-class, idempotency-respecting operation; and dead-letter retention
outlives the source queue's — dead letters are evidence.

## Graceful shutdown

A shutdown signal sets a flag; the polling loop stops receiving; in-flight
work completes within a shutdown timeout; received-but-unstarted messages are
returned immediately for prompt redelivery. The recovery matrix holds with no
lost work: acknowledged work stays done, interrupted work reappears after its
visibility timeout, unstarted work reappears immediately. Multiple loops in
one process — agent workers, evaluation workers, dead-letter consumers —
coordinate through one group that owns signal handling, health probes, and
the watchdog (RFC-0009).

## Verification

Queue semantics are concurrency semantics, and their bugs — finalization
races, lost wakeups — do not reproduce in example-based tests.
Implementations SHOULD maintain machine-checked models of the queue, lease,
and shutdown state machines, plus property tests derived from the same
invariants. Unusual rigor for an "agent library" — and exactly the point of
calling it a control plane.

## Anti-patterns

- **Acknowledge-then-process.** Inverts the loss guarantee; a crash after
  acknowledge silently loses work.
- **Log-and-drop failures.** A failed request that vanishes into a log line
  is invisible to remediation; the dead-letter queue is the failure inbox.
- **Queue as event bus.** Broadcasting telemetry through the work queue, or
  distributing work through the in-process dispatcher (RFC-0006), conflates
  tools with opposite delivery semantics.

## Compatibility surface

Across queue backends, a shared validation suite MUST assert:

- Visibility, finalization, delivery-count, and reply semantics behave
  identically — including expired-receipt and double-finalization errors.
- The dead-letter flow end to end: thresholds, immediate classes, exclusions,
  envelope completeness, source acknowledgment.
- The shutdown recovery matrix under induced interruption at each phase.
