# RFC-0004: Policies — Declarative Invariants over Workflows

- Status: Draft
- Ring: definition
- Depends on: RFC-0002, RFC-0003, RFC-0006

## Summary

Policies are declarative invariants that gate the agent's actions: "a file
must be read before it is overwritten", "tests must have passed before
deployment", "this tool requires that capability". They are declared on the
definition, evaluated before every gated action, composed by conjunction, and
**fail closed with an explanation**. Policies constrain the search space
without scripting the path — the alternative to orchestration graphs, not a
variant of them.

## Motivation

A workflow encodes *how* to accomplish a goal: read, parse, patch, test. When
reality diverges — the file is missing, the test framework is unexpected, the
patch reveals a second bug — a workflow can only fail, skip, or branch, and
each branch multiplies the decision tree the author must foresee and maintain.
The result is a system that is simultaneously rigid and unauditable.

The insight worth building on: for open-ended work, the durable knowledge is
not the sequence but the **invariant**. The author usually cannot know the
right path in advance, but they can state what must hold along any path. An
agent with real reasoning capability, constrained by invariants, finds paths
the author didn't anticipate — and is blocked from the states the author
feared.

| Aspect | Workflow | Policy |
| --- | --- | --- |
| Specifies | Steps to execute | Constraints to satisfy |
| On the unexpected | Fails or branches | Agent reasons within constraints |
| Composition | Sequential coupling | Independent conjunction |
| Agent's role | Executor | Reasoner |

## Design

### The policy contract

A policy is a named object with two duties:

- **`check(action, parameters, context) → decision`** — evaluated before a
  gated action executes. The decision is a boolean **plus a reason**: if
  denied, the reason explains what invariant failed and, ideally, what would
  satisfy it. The reason is delivered to the model as the action's structured
  failure (RFC-0003) and recorded in the ledger.
- **`observe(action, parameters, result, context)`** — called after an action
  **commits** successfully, so the policy can update its tracked state (e.g.,
  "this path has now been read").

Normative properties:

1. **Fail closed.** If a policy cannot decide — missing state, internal error,
   ambiguous input — it MUST deny. An uncertain allow is a silent invariant
   violation; an uncertain deny is a legible bump the agent can reason about.
2. **Explanations are mandatory.** A denial without a reason teaches the model
   nothing and produces thrashing. The reason is part of the contract, not a
   nicety.
3. **Conjunction.** Multiple policies may govern one action; all must allow.
   Each policy MUST be evaluable in isolation — no inter-policy ordering or
   communication.
4. **State in the ledger.** Policy state (what has been invoked, which keys
   have been satisfied) lives in the state ledger (RFC-0006) as typed slices —
   inspectable in run records, restored by transactions, never hidden in
   policy-object fields.
5. **Pure enforcement.** Policies decide and observe; they MUST NOT mutate
   the workspace, invoke tools, or inject instructions. A policy that *does*
   things is orchestration wearing a policy costume.

### Canonical policy shapes

Two shapes cover a large fraction of real needs and belong in any standard
library of policies:

- **Sequential dependency** — action B requires that action A has succeeded
  at least once. Declared as data (a dependency map), not code.
- **Parameter-keyed dependency** — action B on key K requires action A on the
  same K first: read-before-overwrite per path, fetch-before-mutate per
  record. The keyed form is what keeps this a genuine invariant rather than a
  step sequence.

Other recurring shapes: capability requirements ("this tool requires that
credential/resource to be bound"), evidence requirements ("a passing test
event must exist in the ledger newer than the last source change"), and
rate/means limits ("no more than N destructive calls per run").

### The gated action surface

Policies gate **actions**, and the action vocabulary is not limited to the
definition's own tools. Native harness actions — built-in file edits, command
execution, web access, sub-agent spawning — are actions like any other, and
where the harness exposes a pre-action interception point, the adapter MUST
route them through the same policy checks with the same semantics: check
before execution, observe after commit, deny with a reason the model reads.
Mechanically this is invisible to the policy: it sees a named action with
parameters, not a distinction between "bridged" and "native".

Where a harness offers no interception point, the adapter MUST declare the
capability absent (RFC-0012) rather than approximate it — an honestly ungated
native surface is visible in the capability matrix and can be mitigated by
disabling native tools; a silently ungated one is a hole in the invariant.

### Interaction with transactions

Policies see `observe` only for **committed** results — a rolled-back action
never updates policy state, so retries re-face the same gates. Because policy
state lives in the ledger, a transaction rollback (RFC-0003) automatically
rewinds it together with everything else. Implementations MUST preserve this
coupling; policy state tracked outside the ledger desynchronizes on rollback
and produces gates that lie.

### Scope and lifetime

Policies are declared on the definition — at the root or on sections, so a
disabled section's policies vanish with its capabilities (RFC-0002). Policy
state is scoped to the run by default. Cross-run invariants ("this agent
already deployed today") are legitimate but MUST be explicit about their
storage and staleness semantics rather than smuggled in via shared mutable
state.

## When workflows are actually right

Policies are not a religion. A fixed sequence is the honest encoding when the
sequence *is* the specification: protocol handshakes, regulated procedures,
small deterministic pipelines where failure is preferable to adaptation. The
test: if any deviation from the path is by definition an error, write the
workflow (or a sequential-dependency policy that pins the whole chain). If
deviation might be intelligence, write invariants.

## Anti-patterns

- **Workflow in policy clothing.** A dependency chain A→B→C→D→E that admits
  exactly one path has all the brittleness of a workflow plus indirection.
- **Over-constraining.** If the conjunction of policies leaves one valid
  action at each step, the agent is an executor again; delete the agent or
  delete the policies.
- **Advisory policies.** A "policy" whose denial the system sometimes ignores
  is feedback (RFC-0005) misfiled; the distinction between hard gates and
  soft guidance must stay crisp or neither is trustworthy.
- **Reasons written for developers.** Denial text full of internal jargon
  wastes the mechanism; the primary reader is the model, mid-run.

## Compatibility surface

An adapter certification suite MUST assert:

- A gated action denied by policy does not execute (no handler effects), and
  the model receives the denial reason as a structured failure.
- Where native-action gating is declared, a policy denial prevents the native
  action's effects and the reason reaches the model, exactly as for
  definition tools.
- Policy state updates only on committed results; a rolled-back action leaves
  policy state unchanged.
- The same definition produces the same allow/deny decisions for the same
  action sequence on every harness.
- Policy decisions (including reasons) appear in the run record.
