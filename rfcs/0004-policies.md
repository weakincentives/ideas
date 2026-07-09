# RFC-0004: Policies — Declarative Invariants over Workflows

- Status: Draft
- Ring: definition
- Depends on: RFC-0002, RFC-0003, RFC-0006

## Summary

Policies are declarative invariants that gate the agent's actions: "a file
must be read before it is overwritten", "tests must have passed before
deployment". They are declared on the definition, checked before every gated
action, composed by conjunction, and **fail closed with an explanation**.
Policies constrain the search space without scripting the path — the
alternative to orchestration graphs, not a variant of them.

## Motivation

A workflow encodes *how*: read, parse, patch, test. When reality diverges, a
workflow can only fail, skip, or branch — and every branch multiplies the
decision tree the author must foresee. For open-ended work the durable
knowledge is not the sequence but the **invariant**: the author cannot know
the right path in advance, but can state what must hold along any path. An
agent constrained by invariants finds paths the author didn't anticipate and
is blocked from the states the author feared.

| Aspect | Workflow | Policy |
| --- | --- | --- |
| Specifies | Steps to execute | Constraints to satisfy |
| On the unexpected | Fails or branches | Agent reasons within constraints |
| Composition | Sequential coupling | Independent conjunction |
| Agent's role | Executor | Reasoner |

## The contract

A policy is a named object with two duties:

- **check(action, parameters, context) → decision** — before a gated action
  executes. The decision is a boolean **plus a reason**: what invariant
  failed and, ideally, what would satisfy it. The reason reaches the model as
  the action's structured failure (RFC-0003) and lands on the transcript.
- **observe(action, parameters, result, context)** — after an action commits,
  so the policy can update its tracked state.

Normative properties:

1. **Fail closed.** A policy that cannot decide MUST deny. An uncertain allow
   is a silent invariant violation; an uncertain deny is a legible bump the
   agent can reason about.
2. **Explanations are mandatory.** A denial without a reason teaches the
   model nothing and produces thrashing.
3. **Conjunction.** Multiple policies may govern one action; all must allow,
   and each MUST be evaluable in isolation.
4. **State on the transcript.** Policy state lives in state slices (RFC-0006)
   — inspectable in run records, rewound by transaction rollback, never
   hidden in policy-object fields. A rolled-back action never updates policy
   state, so retries re-face the same gates.
5. **Pure enforcement.** Policies decide and observe; they MUST NOT mutate
   the workspace, invoke tools, or inject instructions. A policy that *does*
   things is orchestration wearing a policy costume.

## Canonical shapes

Two shapes cover most real needs: **sequential dependency** (action B
requires that A has succeeded, declared as data) and **parameter-keyed
dependency** (B on key K requires A on the same K — read-before-overwrite per
path). The keyed form is what keeps this an invariant rather than a step
sequence. Other recurring shapes: capability requirements, evidence
requirements ("a passing-test event newer than the last source change"), and
means limits ("no more than N destructive calls per run").

## The gated action surface

Policies gate **actions**, not just the definition's own tools. Native
harness actions — built-in file edits, command execution, web access,
sub-agent spawning — are actions like any other. Where the harness exposes a
pre-action interception point, the adapter MUST route them through the same
checks with the same semantics: check before, observe after commit, deny with
a reason the model reads. The policy sees a named action with parameters, not
a "bridged"/"native" distinction.

Where a harness offers no interception point, the adapter MUST declare the
capability absent (RFC-0012) rather than approximate it. An honestly ungated
native surface is visible and can be mitigated by disabling native tools; a
silently ungated one is a hole in the invariant.

## Scope

Policies are declared at the definition root or on sections, so a disabled
section's policies vanish with its capabilities (RFC-0002). Policy state is
run-scoped by default; cross-run invariants are legitimate but MUST be
explicit about storage and staleness rather than smuggled in through shared
mutable state.

## When workflows are right

A fixed sequence is the honest encoding when the sequence *is* the
specification: protocol handshakes, regulated procedures, pipelines where
failure beats adaptation. The test: if any deviation is by definition an
error, write the workflow (or pin the chain with a dependency policy). If
deviation might be intelligence, write invariants.

## Anti-patterns

- **Workflow in policy clothing.** A dependency chain admitting exactly one
  path has a workflow's brittleness plus indirection.
- **Over-constraining.** If policies leave one valid action at each step, the
  agent is an executor again — delete the agent or delete the policies.
- **Advisory policies.** A denial the system sometimes ignores is feedback
  (RFC-0005) misfiled; the hard/soft line must stay crisp or neither is
  trustworthy.
- **Reasons written for developers.** The primary reader of a denial is the
  model, mid-run.

## Compatibility surface

An adapter certification suite MUST assert:

- A denied action does not execute, and the model receives the reason as a
  structured failure — for definition tools always, and for native actions
  where the capability is declared.
- Policy state updates only on committed results; rollback leaves it
  unchanged.
- The same definition yields the same allow/deny decisions for the same
  action sequence on every harness.
- Policy decisions and reasons appear in the run record.
