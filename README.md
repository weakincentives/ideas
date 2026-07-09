# The Agent-Definition Layer

## 1. Thesis

Every production agent has two halves: the **definition** and the
**harness**. The definition is what the agent *is* — its instruction
structure, tools, policies, state contract, and completion criteria. The
harness is what the runtime *does* — model calls, planning loops, sandboxing,
tool dispatch, scheduling, recovery.

For the first generation of agent software these halves were built together,
and that was tolerable while the harness was a thin loop around a model
endpoint. It no longer is. Harnesses are serious runtime products, improving
faster than application teams can track, and the workspace an agent operates
on is increasingly an ephemeral, sandboxed, or remote environment the
definition should know nothing about. Teams that fuse definition and harness
rewrite their agents every time a better harness appears.

The bet is simple: **the harness is the depreciating asset; the definition is
the durable one.** The agent-definition layer makes the definition a
portable, reviewable, typed artifact, carried to whichever harness fits by a
thin, certified adapter — and wraps the delegation in a control plane that
makes it safe to walk away: bounded execution, durable work distribution, a
complete run record, versioned iteration, and evaluation.

In one line: one artifact defines the agent, one stream records the run,
explicit bounds contain it, tests prove it ports, and evaluation improves it.

This document states the position. The [RFC collection](rfcs/README.md)
turns it into specific, testable contracts.

## 2. Principles

1. **Own the definition; rent the harness.** The definition is the durable
   asset. Planning loops, sandboxes, and recovery are runtime concerns —
   important, and replaceable.

2. **Make the correct path the easy path.** Shape incentives before enforcing
   constraints: instructions co-located with the tools they document, typed
   contracts that make invalid payloads unrepresentable, progressive
   disclosure that keeps attention on what matters now, explicit state the
   model can reason over. Gates are the backstop, not the strategy.

3. **Declare invariants, not paths.** Encode what must hold, not the steps to
   follow. For open-ended work, procedural orchestration suppresses the
   reasoning that justified using an agent; policies constrain the search
   space while leaving the path to the agent.

4. **Fail closed; surface reasons.** When a policy is uncertain, deny. When
   completion is unverified, block. When an environment half-materializes,
   tear it down. Always with a reason the agent can act on and an operator
   can audit — silent degradation is the enemy of unattended operation.

5. **One artifact.** Nothing the agent can see or do exists outside the
   rendered definition. No registry to synchronize, no configuration that
   changes capabilities outside review: rendering the definition *is* the
   capability set.

6. **One stream.** Everything that happens is an event on the transcript —
   one append-only stream per run. State, conversation, and metrics are views
   of it; new audiences get new views, not new logs. History cannot be
   erased, only views rewound.

7. **Effects are transactions.** Every tool call commits atomically or leaves
   no trace in state or workspace. Failures return to the model as structured
   results the run can survive.

8. **Nothing ambient.** Environment, capabilities, credentials, and time are
   declared and injected. Egress is default-deny; secrets are referenced by
   name and bound outside the agent's reach; direct clock reads are
   prohibited. A definition's behavior is a function of its declared inputs.

9. **Everything bounded; liveness is proof of work.** Every run has a
   deadline and budget enforced at checkpoints; sub-agent spend is parent
   spend. Leases extend only on heartbeats from real work — a stuck agent
   forfeits its work to a healthy peer by default.

10. **Portability is proven, not asserted.** Adapters are thin, prohibited
    from reinterpreting the definition, and certified by one compatibility
    suite with the transcript as its oracle. An honest capability gap is
    compatible; a quiet approximation is not.

11. **Change is an experiment.** Overrides bind to content hashes and retire
    when the underlying text changes; variants ship as immutable experiments;
    promotion is won by evaluation on the production path — never by vibes,
    never by config forks.

12. **Runs explain themselves.** Every run produces a self-contained,
    queryable record for a reader who wasn't there — increasingly another
    agent. The record is the explanation, not a pile of logs.

## 3. The Problem

There is no widely accepted artifact that represents "the agent"
independently of "the agent running on this harness." What teams call the
agent is usually a fused tangle: instructions in one file, a tool registry
assembled elsewhere, permissions in runtime glue, completion checks bolted
on, state spread across mutable objects, local-filesystem assumptions
throughout. The failure modes are concrete:

- **Accidental lock-in.** The agent cannot move because definition and
  runtime are fused at every layer.
- **Workflow brittleness.** Procedural agents fail when reality diverges from
  the path, or accumulate decision trees harder to audit than the work.
- **Premature termination.** Unattended agents declare success with work
  remaining, and nothing blocks the stop or explains what is missing.
- **Untestable behavior.** Regression testing requires standing up the full
  harness because no smaller artifact captures the agent's contract.
- **Filesystem coupling.** Tools written against local assumptions break in
  sandboxed, remote, or virtual workspaces.
- **Operational opacity.** A run's history is a pile of logs rather than an
  explanation of what happened and why.
- **Adapter drift.** "Same definition on a different harness" decays into
  "similar behavior if nothing unusual happens."

The root cause is the same in every case: no first-class definition layer
with a compatibility contract.

## 4. The Architecture

Three rings, specified by the [RFCs](rfcs/README.md):

**The definition** (RFCs 0002–0006) is the portable artifact, in five parts.
The *instruction graph* bundles prose with the capabilities it documents;
rendering it deterministically produces both the text the model sees and the
exact tool set the agent has. *Tools* are the typed, transactional
side-effect boundary. *Policies* are fail-closed invariants gating every
action — including the harness's native tools. *Feedback and completion*
steer the trajectory and gate termination against the definition's own
success criteria. The *transcript and state* contract records every event on
one append-only stream and folds typed state views from it.

**The control plane** (RFCs 0007–0014) is the set of contracts that make
unattended delegation safe, independent of which harness provides the
machinery: environments declared as data with default-deny egress; explicit
capabilities and injected time; the execution envelope of deadlines, budgets,
and proof-of-work leases; at-least-once work distribution with dead-letter
forensics; self-contained queryable run records; hash-anchored overrides and
immutable experiments; and evaluation running on the production path — the
gate through which every change is promoted.

**The harness** executes: model calls, planning loop, native tools, sandbox
technology, workspace provisioning. The *adapter* (RFC 0012) maps the
definition onto it and may not reinterpret it; one compatibility suite,
gated by honest capability declarations, certifies that the mapping preserved
the contracts.

## 5. What Success Looks Like

A team writes an unattended agent once and tests its policies, tools,
completion gates, reducers, and rendering without launching a harness or
calling a model. They run it locally for iteration, then on a remote sandbox
for unattended execution, then compare a new harness when the tradeoffs
change — the definition does not change; only the adapter and configuration
do.

A new harness ships an adapter; it passes the compatibility suite; existing
definitions run there. A new model version ships; the team submits its
regression dataset under baseline and treatment experiments, compares pass
rates and behavioral assertions, and promotes the winner — an ordinary change
with a gate, not a leap of faith.

A new engineer reads one definition to learn what the agent may do and what
counts as done. An auditor asks what the agent can touch and why it stopped;
the answer is the rendered definition and the run record, not a
reconstruction from runtime code and log streams. The center of gravity moves
from runtime plumbing to definition design — instruction structure, tool
surface, policy set, completion criteria — which is where the durable value
was all along.

## 6. The RFCs

| RFC | Title |
| --- | --- |
| [0001](rfcs/0001-scope-and-boundary.md) | Scope and the Boundary |
| [0002](rfcs/0002-instruction-graph.md) | The Instruction Graph |
| [0003](rfcs/0003-tools.md) | Tools: The Transactional Side-Effect Boundary |
| [0004](rfcs/0004-policies.md) | Policies: Declarative Invariants over Workflows |
| [0005](rfcs/0005-feedback-and-completion.md) | Feedback and Completion Gates |
| [0006](rfcs/0006-transcript-and-state.md) | The Transcript and State |
| [0007](rfcs/0007-workspace.md) | Workspace, Sandbox, and Egress |
| [0008](rfcs/0008-capabilities-and-time.md) | Capabilities, Resources, and Injected Time |
| [0009](rfcs/0009-execution-envelope.md) | The Execution Envelope |
| [0010](rfcs/0010-work-distribution.md) | Work Distribution |
| [0011](rfcs/0011-run-record.md) | The Run Record |
| [0012](rfcs/0012-adapter-contract.md) | Adapters and the Compatibility Suite |
| [0013](rfcs/0013-versioning-and-overrides.md) | Versioned Iteration: Overrides and Experiments |
| [0014](rfcs/0014-evaluation.md) | Evaluation as the Control Loop |

## 7. Appendix: The Boundary Diagram

```text
+------------------------------------------------------------------+
| DEFINITION — authored, versioned, portable                       |
|                                                                  |
|   instruction graph · tools · policies                           |
|   feedback & completion gates · transcript & state               |
|                                                                  |
|   Reviewable and testable without a harness.                     |
+------------------------------------------------------------------+
| CONTROL PLANE — contracts for unattended delegation              |
|                                                                  |
|   workspace & egress · capabilities & time · envelope            |
|   work distribution · run records · overrides & experiments      |
|   evaluation · adapter certification                             |
+-------------------------------+----------------------------------+
                                |
                                | Adapter — thin, certified,
                                | forbidden to reinterpret
                                v
+------------------------------------------------------------------+
| HARNESS — rented, swappable                                      |
|                                                                  |
|   planning loop & model calls · native tools · sandboxing        |
|   workspace provisioning · transport & streaming · recovery      |
+------------------------------------------------------------------+
```

The lines between the boxes are the architectural commitment. Everything else
is implementation detail.
