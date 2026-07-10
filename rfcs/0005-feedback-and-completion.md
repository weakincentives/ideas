# RFC-0005: Feedback and Completion Gates

- Status: Draft
- Ring: definition
- Depends on: RFC-0002, RFC-0003, RFC-0006, RFC-0009

## Summary

Unattended agents drift, and unattended agents stop early. The definition
answers both: **feedback providers** observe the trajectory and inject
advisory guidance mid-run when triggers fire; **completion gates** block
termination until the definition's own success criteria hold, returning
actionable feedback about what is missing. Feedback steers without gating;
completion gates gate exactly one thing — the decision to stop.

## Motivation

In attended use, the human is the feedback loop: noticing circles, recalling
the deadline, refusing half-finished work. Remove the human and the naive
fixes fail — anticipatory guidance bloats the initial context and cannot
react; outer retry loops discard the run's accumulated context; and trusting
the model's own "I'm done" is the single most common failure of unattended
agents. Guidance must arrive *inside* the run, at the moment it is relevant,
and "done" must be a verifiable claim.

## Feedback providers

A provider declares **triggers** (every N tool calls, every T seconds, on an
observable milestone such as an artifact appearing — OR'd, with per-provider
cadence), an optional relevance check, and a **produce** step that inspects
the trajectory through read views of state, workspace, and envelope.

Semantics:

1. **Advisory, not binding.** Feedback is injected into the model's context;
   the agent decides. Anything that must be enforced is a policy (RFC-0004).
2. **Immediate delivery.** Feedback produced after a tool call is injected
   into the current turn — not deferred to the next render — so
   self-correction happens while the provoking context is fresh.
3. **All matching providers run.** Simultaneous triggers all fire and deliver
   together, each attributed to its provider. First-match-wins silently drops
   guidance.
4. **Recorded.** Every delivery lands on the ledger — this is how
   per-provider cadence works and how run records show what the agent was
   told and when.
5. **Marked.** Feedback is delivered as feedback, attributed, so the model
   distinguishes steering from task content and the transcript stays
   parseable.

Standard providers worth shipping: a deadline reporter with escalating
urgency, a one-shot milestone notifier, a cadence reminder. The valuable ones
are definition-specific: "you have edited N files but run no tests."

## Completion gates

A gate is consulted when the agent or harness proposes to stop. Input: the
tentative output, read views of state and workspace, and the stated stop
reason. Output: *complete*, or *incomplete with feedback* — a concrete
statement of what is missing.

Semantics:

1. **Blocking.** Incomplete prevents termination; the feedback is injected
   and the agent continues. This is the one place the definition overrides
   the model's judgment of its own work.
2. **Bounded.** Continuation rounds are capped. A gate that never passes
   surfaces as a failed run carrying the gate's last feedback — not an
   infinite loop.
3. **Envelope bypass.** When deadline or budget is exhausted (RFC-0009),
   gates are bypassed and the run ends with best-effort output plus a
   recorded note that completion was not verified. Forcing turns with no
   budget is strictly worse than stopping; the bypass itself is recorded.
4. **Composable.** AND/OR composition makes "report exists AND (tests pass OR
   waiver recorded)" declarable without custom code.
5. **Verifiable claims only.** Gates check evidence — files that exist,
   events that occurred, output that parses and satisfies predicates — not
   restated instructions. "Did you do a good job?" is not a gate; expensive
   judgment belongs in evaluation (RFC-0014), off the critical path.

## Division of labor

| Mechanism | Binds? | Scope | Question |
| --- | --- | --- | --- |
| Policy (RFC-0004) | Hard gate | One action | "May this happen?" |
| Feedback | Advisory | Trajectory | "Are we on track?" |
| Completion gate | Hard gate | Termination | "Is the work done?" |

All three live on the definition because all three describe the work, not the
runtime. Porting the agent ports its steering and its definition of done.

## Anti-patterns

- **Nagging.** High-frequency feedback trains the model to ignore feedback.
  Fire on signal, stay silent when the trajectory is healthy.
- **Feedback as covert control.** Commands phrased as feedback to simulate
  enforcement — if it must happen, gate it.
- **Unverifiable gates.** Re-asking the model to self-assess adds a round
  trip and no assurance.
- **Bypass laundering.** A record that cannot distinguish "verified complete"
  from "stopped at the bell" falsifies the run.

## Compatibility surface

An adapter certification suite MUST assert:

- Triggered feedback reaches the live run (observable in the transcript) and
  is recorded with provider attribution.
- An incomplete verdict blocks the stop and its feedback reaches the model; a
  complete verdict allows it.
- Envelope bypass occurs on exhaustion and is recorded as bypass.
- Composite gate logic yields the same decisions on every harness.
- The continuation cap is enforced.
