# RFC-0014: Evaluation as the Control Loop

- Status: Draft
- Ring: control plane
- Depends on: RFC-0006, RFC-0010, RFC-0011, RFC-0013

## Summary

Evaluation is the loop that makes every other change safe — prompt overrides,
definition refactors, harness upgrades, model migrations. The contract:
**datasets** of typed samples; **evaluators** as pure functions over outputs
*and over the run's state* (behavioral assertions, not just answer-matching);
evaluation runs executed through the **same production path** as real
traffic; **experiments** (RFC-0013) as the unit of comparison; and, because
volume outruns human attention, **analysis agents** that investigate run
records automatically.

## Motivation

The compatibility suite (RFC-0012) proves an adapter preserved mechanics;
only evaluation proves the agent still does its job well. Without it, every
model upgrade, harness move, and prompt tweak is a leap of faith taken in
production; with it, they are ordinary changes with regression gates. Two
agent realities shape the design: quality is only partly about final answers
— *how* the agent worked is often the regression that matters — and agent
evaluation is expensive, so it must scale on the same distribution substrate
the fleet already runs (RFC-0010).

## Datasets and evaluators

A sample is `(id, input, expected)` with typed fields; a dataset is an
immutable, loadable collection with stable ids so comparisons join across
runs. Datasets accumulate: **every interesting production failure — dead
letters, anomalous run records — is a candidate sample.** That pipeline is
how an unattended fleet learns from its incidents.

An evaluator is a pure function `(output, expected) → score`, a score
carrying a normalized value, a pass/fail verdict, and a reason; evaluators
compose with all-of/any-of logic.

**Behavioral evaluators** additionally receive read access to the run's state
views (RFC-0006) and assert on trajectory: a tool was or wasn't called, call
counts fall in range, no tool failed, token usage stayed under a bound, a
predicate holds over a typed slice. This is the payoff of the ledger
contract — behavior is data, so behavior is assertable with zero added
instrumentation. Without it, degenerate passes go undetected: an agent that
hardcodes the expected answer or skips verification looks identical to a good
one.

**Judge evaluators** use a model against a rubric with a small calibrated
ordinal scale and a pass threshold, the criterion text versioned with the
evaluation config. Judges are evaluators like any other — composed behind
cheap objective gates, never a monoculture.

## The production path

An evaluation executes the definition through the same loop, adapter,
envelope, and run-record machinery as production traffic — the eval loop
wraps the production loop rather than replacing it. Requests arrive as
`(sample, experiment)` pairs on a queue; results return with score, latency,
error, and a link to the per-sample run record. Consequences: what is
measured is measured *with* the guardrails and transactional semantics of
production, not an idealized bench; evaluation scales horizontally like any
other work, and poisoned samples dead-letter instead of wedging the sweep;
and every scored sample has a full run record, so "why did this fail?" has
the same answer path as a production incident. Optimizations such as caching
are permitted only where they provably do not change what is measured —
"measure what ships" is the line.

## Experiments and comparison

The experiment is the unit of comparison: one dataset submitted under
baseline and treatments; results grouped by experiment; pass rates, scores,
and latency compared with deltas. Reports MUST keep per-sample results
addressable so regressions decompose into named, replayable cases, and SHOULD
surface sample counts alongside every comparison — small eval sets breed
overconfident promotions. This machinery serves rollouts as well as research:
a new model, harness version, or override tag is a treatment; promotion means
it beat baseline on the regression dataset; and the promoted variant's
identity is in every subsequent run record (RFC-0013).

## Analysis agents

A fleet produces more records than any team reads. The final stage is
automated analysis:

- Execution loops emit **completion notifications** (source, run-record
  reference, outcome, score) to a queue — fire-and-forget; producers know
  nothing about analysis.
- A **forwarder** applies sampling and budget: always forward failures,
  sample successes at a low rate, stop at a request budget per window.
  Analysis must never out-spend the work it analyzes.
- The **analysis loop** is itself an ordinary agent on this architecture,
  whose tool surface is the run-record query interface (RFC-0011) and whose
  objective is stated per deployment.

Analysis agents observe; they do not act. Their deliverable is a
**structured finding**: a typed report carrying the conclusion, evidence
references (run-record identities and the queries supporting the claim), and
— where a remedy is obvious — machine-actionable proposals: a candidate
override (RFC-0013), a candidate regression sample, a suggested error
classification (RFC-0010). Proposals earn adoption through the normal gates;
a proposed override is promoted by evaluation like any other change.
Structure makes findings cheap to act on; provenance makes acting on them
safe.

The best debugger for a complex agent is another agent with a query tool and
a stable schema — a claim that only holds because of the rest of the
collection: complete records, typed state, attributable variants.

## Anti-patterns

- **The parallel bench.** An eval harness with its own execution path
  measures a system that doesn't ship.
- **Answer-only scoring.** Degenerate behavior passes undetected without
  behavioral evaluators.
- **Eval-set rot.** Datasets that never ingest production failures measure
  the agent against last year's world.
- **Unbudgeted analysis.** Meta-agents analyzing every run recreate the cost
  problem they were meant to manage.

## Compatibility surface

An adapter certification suite MUST assert:

- The evaluation path produces per-sample run records structurally identical
  to production records on every harness.
- Behavioral evaluators read the same state shapes across harnesses,
  end to end.
- Experiment identity flows from request to result to run record unchanged.
