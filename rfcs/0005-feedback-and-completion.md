# RFC-0005: Feedback and Completion Gates

- Status: Draft
- Ring: definition
- Depends on: RFC-0002, RFC-0003, RFC-0006, RFC-0009

## Summary

Unattended agents drift, and unattended agents stop early. The definition
answers both with two mechanisms that belong beside its tools and policies:
**feedback providers** — trajectory observers that inject advisory guidance
mid-run when trigger conditions fire — and **completion gates** — checks that
block termination until the definition's own success criteria are satisfied,
returning actionable feedback about what is missing. Feedback steers without
gating; completion gates gate exactly one thing: the decision to stop.

## Motivation

In attended use, the human is the feedback loop: they notice the agent going
in circles, remind it of the deadline, and refuse to accept half-finished
work. Removing the human removes all three corrections. The naive fixes fail:

- Cramming anticipatory guidance into the initial instructions bloats context
  and cannot react to what actually happens.
- Wrapping the agent in an outer retry loop ("if output bad, re-prompt")
  discards the run's accumulated context and converts drift into cost.
- Trusting the model's own "I'm done" is the single most common failure of
  unattended agents: premature success declarations with work remaining.

The mechanisms below deliver guidance *inside* the run, at the moment it is
relevant, and make "done" a verifiable claim rather than a vibe.

## Design

### Feedback providers

A feedback provider is a named observer declared on the definition with:

- **Triggers** — conditions over the trajectory that decide *when* it runs:
  every N tool calls, every T seconds, or on an observable milestone (e.g., a
  particular artifact appearing in the workspace). Trigger kinds are OR'd;
  each provider tracks its own cadence independently.
- **A relevance check** — an optional finer filter once triggered.
- **A produce step** — inspects the trajectory (via a read view of the state
  ledger, the workspace, and the envelope) and returns guidance text.

Normative semantics:

1. **Advisory, not binding.** Feedback is injected into the model's context;
   the agent decides how to respond. Anything that must be *enforced* is a
   policy (RFC-0004), not feedback. Implementations MUST keep this boundary.
2. **Immediate delivery.** Feedback produced after a tool call is injected
   into the current turn — not deferred to the next render — so
   self-correction happens while the context that provoked it is fresh.
3. **All matching providers run.** When several trigger simultaneously, all
   are evaluated and their outputs delivered together, each clearly
   attributed to its provider. First-match-wins silently drops guidance.
4. **Recorded.** Every delivered feedback is stored in the ledger — this is
   both how per-provider cadence works and how run records can show what the
   agent was told and when.
5. **Structured delivery.** Feedback is marked in context as feedback with
   its provider name, so the model can distinguish steering from task content
   and the transcript stays parseable.

Canonical providers any implementation should ship: a **deadline reporter**
(remaining time, escalating urgency near exhaustion), a **milestone
notifier** (one-shot guidance when an expected artifact appears), and a
**static reminder** on a cadence. The interesting ones are
definition-specific: "you have edited N files but run no tests", "your last
five calls repeated the same query".

### Completion gates

A completion gate is a check declared on the definition and consulted when the
agent (or harness) proposes to stop:

- **Input**: the tentative output, a read view of the ledger, the workspace,
  and the stated stop reason.
- **Output**: *complete*, or *incomplete with feedback* — a concrete,
  actionable statement of what is missing ("required file X does not exist",
  "the deliverable names three issues but the ledger shows only one
  verified").

Normative semantics:

1. **Blocking.** An incomplete verdict prevents termination; the feedback is
   injected and the agent continues. This is the one place the definition
   overrides the model's judgment about its own work.
2. **Bounded.** Continuation rounds MUST be capped. A gate that never passes
   must eventually surface as a failed run with the gate's last feedback in
   the record — not an infinite loop.
3. **Envelope bypass.** When the deadline or budget is exhausted (RFC-0009),
   gates MUST be bypassed and the run ends with best-effort output plus a
   record that completion was not verified. Forcing more turns with no budget
   is strictly worse than stopping; the bypass MUST itself be recorded.
4. **Composable.** Gates compose with AND/OR logic so "report exists AND
   (tests pass OR waiver recorded)" is declarable without custom code.
5. **Verifiable claims only.** A gate SHOULD check evidence — files that
   exist, events in the ledger, checks that pass — not restate instructions.
   A gate that asks "did you do a good job?" is prompt engineering, not
   verification.

The most valuable gates are cheap and objective: required output files exist;
required ledger events occurred; structured output parses and satisfies
domain predicates. Expensive judgment (was the analysis *good*?) belongs in
evaluation (RFC-0014), which runs off the critical path.

### Division of labor

| Mechanism | Binds? | Scope | Question answered |
| --- | --- | --- | --- |
| Policy (RFC-0004) | Hard gate | One action | "May this happen?" |
| Feedback | Advisory | Trajectory | "Are we still on track?" |
| Completion gate | Hard gate | Termination | "Is the work actually done?" |

All three live on the definition because all three describe the work, not the
runtime. Porting the agent to a new harness ports its steering and its
definition of done.

## Anti-patterns

- **Nagging.** High-frequency feedback trains the model to ignore feedback.
  Triggers should fire on signal, not schedule, and providers should stay
  silent when the trajectory is healthy.
- **Feedback as covert control.** Phrasing commands as feedback ("STOP and do
  X NOW") to simulate enforcement — if it must happen, gate it.
- **Unverifiable gates.** Gates that re-ask the model to self-assess add a
  round trip and no assurance.
- **Ungated envelope exhaustion.** Treating deadline bypass as gate passage
  in the run record; the record must distinguish "verified complete" from
  "stopped at the bell".

## Compatibility surface

An adapter certification suite MUST assert:

- Triggered feedback is delivered into the live run (observable in the
  transcript) and recorded in the ledger with provider attribution.
- An incomplete gate verdict blocks a stop and its feedback reaches the
  model; a complete verdict allows the stop.
- Gate bypass on deadline/budget exhaustion occurs and is recorded as bypass.
- Composite gate logic (AND/OR) produces the same decisions on every harness.
- The continuation cap is enforced.
