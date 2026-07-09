# RFC-0014: Evaluation as the Control Loop

- Status: Draft
- Ring: control plane
- Depends on: RFC-0006, RFC-0010, RFC-0011, RFC-0013

## Summary

Evaluation is not a side activity; it is the control loop that makes every
other kind of change safe — prompt overrides, definition refactors, harness
upgrades, model migrations. The contract: **datasets** of typed samples;
**evaluators** as pure functions over outputs *and over the state ledger*
(behavioral assertions, not just answer-matching); evaluation runs that
execute through the **same production execution path** (same loops, queues,
envelopes, run records) rather than a parallel test harness; **experiments**
(RFC-0013) as the comparison unit; and — because eval volume outruns human
attention — **analysis agents** that investigate run records automatically.

## Motivation

Teams that fuse definition and harness rewrite their agents when the harness
changes. Teams that separate them (RFC-0001) still face the question: *how do
you know the agent survived the move?* The compatibility suite (RFC-0012)
proves the adapter preserved mechanics; only evaluation proves the agent
still does its job well. The same holds for every model upgrade and every
prompt tweak. Without an evaluation loop, each of those is a leap of faith
taken in production; with one, they are ordinary changes with regression
gates.

Two agent-specific realities shape the design. First, agent quality is only
partly about final answers — *how* the agent worked (which tools, how many
calls, how much budget, whether it verified before claiming) is often the
regression that matters, and answer-matching alone misses it. Second,
agent evaluation is expensive and slow, so it must scale out on exactly the
work-distribution substrate the fleet already has (RFC-0010), and its
failures deserve the same forensics (RFC-0011).

## Design

### Datasets and samples

A sample is `(id, input, expected)` with typed input and expectation;
a dataset is an immutable, loadable collection of them. Nothing exotic —
the important properties are that samples are versionable data, that ids are
stable (comparisons across runs join on them), and that datasets accumulate:
**every interesting production failure (RFC-0010's dead letters, RFC-0011's
run records) is a candidate sample.** The DLQ-to-dataset pipeline is how an
unattended fleet learns from its incidents.

### Evaluators

An evaluator is a pure function `(output, expected) → score`, where a score
carries a normalized value, a pass/fail verdict, and a human-readable reason.
Composition is first-class: all-of (mean/AND), any-of (max/OR).

**Behavioral evaluators** additionally receive a read view of the run's state
ledger (RFC-0006) and assert on trajectory: a given tool was (or was never)
called; call counts fall in a range; no tool call failed; token usage stayed
under a bound; an arbitrary predicate holds over a typed slice. This is the
payoff of the event-sourced state contract — the run's *behavior* is data, so
behavior is assertable with zero instrumentation added to the definition.

**Judge evaluators** use a model to grade against a rubric with a small
calibrated ordinal scale (e.g., five steps mapping to fixed values and a
pass threshold), with the criterion text part of the versioned evaluation
config. Judges are evaluators like any other — composable with exact checks,
so cheap objective assertions gate before expensive subjective ones.

### Evaluation runs on the production path

An evaluation executes the definition through the same loop, adapter,
envelope, and run-record machinery as production traffic — the eval loop
*wraps* the production loop rather than replacing it. Requests arrive as
`(sample, experiment)` pairs on a queue; results return with score, latency,
error, and a link to the per-sample run record. Consequences that are
normative:

- Whatever is measured, is measured **with** the guardrails, envelopes, and
  transactional semantics of production — not an idealized bench.
- Evaluation scales horizontally like any other work (RFC-0010), and poisoned
  samples dead-letter instead of wedging the sweep.
- Every scored sample has a full run record, so "why did this fail?" has the
  same answer path as production incidents.

### Experiments and comparison

The experiment (RFC-0013) is the unit of comparison: submit one dataset under
baseline and treatments; group results by experiment; report pass rates, mean
scores, latency; compare treatment to baseline with deltas and relative
improvement. Reports MUST keep per-sample results addressable (not just
aggregates) so regressions decompose into named, replayable cases.

This machinery serves rollouts as well as research: a new model, harness
version, or override tag is an experiment; promotion is "the treatment beat
baseline on the regression dataset"; and the promoted variant's identity is
in every subsequent run record.

### Analysis agents

A fleet's evaluation and production runs produce more records than any team
reads. The final stage of the control loop is automated analysis:

- Execution loops emit **completion notifications** (source, run-record
  reference, success, score) to a queue as they finish — fire-and-forget;
  producers know nothing about analysis.
- A **forwarder** applies sampling and budget policy: always forward
  failures, sample successes at a low rate, stop at a request budget per
  window. Analysis must never out-spend the work it analyzes.
- An **analysis loop** is itself an agent (an ordinary definition on this
  same architecture) whose tool surface is the run-record query interface
  (RFC-0011), tasked with an objective ("why do failures cluster on samples
  with long inputs?") and producing analysis reports — which are themselves
  run records.

The best debugger for a complex agent is another agent with a query tool and
a stable schema. That claim only holds because of the rest of the collection:
complete records (0011), typed state (0006), attributable variants (0013).

## Anti-patterns

- **The parallel bench.** An eval harness with its own execution path
  measures a system that doesn't ship.
- **Answer-only scoring.** Passing outputs from degenerate behavior (agent
  hardcodes the expected string, skips verification) go undetected without
  behavioral evaluators.
- **Judge monoculture.** A single LLM judge with an uncalibrated scale as the
  only signal; judges drift, and objective gates must anchor them.
- **Eval-set rot.** Datasets that never ingest production failures measure
  the agent against last year's world.
- **Unbudgeted analysis.** Meta-agents analyzing every run recreate the cost
  problem they were meant to manage.

## Compatibility surface

An adapter certification suite MUST assert (thin here by design — most of
this RFC binds the library):

- The evaluation path produces per-sample run records identical in structure
  to production records, on every harness.
- Behavioral evaluators read the same ledger shapes across harnesses (follows
  from RFC-0006 parity, asserted end-to-end here).
- Experiment identity flows from request to result to run record unchanged.

## Open questions

- Statistical rigor: should the comparison contract include confidence
  intervals and minimum-N warnings natively, or is that reporting-layer
  concern? Small eval sets breed overconfident promotions.
- Caching: repeated identical (sample, definition-version) executions are
  wasteful, but caching model behavior undermines "measure what ships" —
  where is the line?
- Should analysis agents be able to *act* (file issues, propose override
  payloads per RFC-0013) or only report? Proposal-only is the safe default;
  the promotion gate stays with evaluation.
