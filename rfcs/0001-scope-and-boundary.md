# RFC-0001: Scope and the Boundary

- Status: Draft
- Ring: framing

## Summary

One bet organizes everything here: **the harness is the depreciating asset;
the definition is the durable one.** The definition — what the agent *is* —
is authored, versioned, reviewed, and tested independently of the harness
that runs it. Between them sits a thin, certified adapter. Around them sits a
control plane: the contracts that make it safe to hand a definition to a
harness and walk away.

The narrative in one line: one artifact defines the agent, one stream records
the run, explicit bounds contain it, tests prove it ports, and evaluation
improves it.

## Why unattended changes the design

1. **Nobody is watching.** No human notices drift, premature success, runaway
   spend, or a hung process. Every correction must be encoded in advance and
   enforced mechanically.
2. **The runtime is not yours.** Harnesses improve faster than application
   teams can track them. Fusing the agent to one converts a product choice
   into lock-in.
3. **The workspace is not local.** Agents act on ephemeral, sandboxed, remote
   environments. Assumptions about local files, ambient credentials, or open
   networks do not survive production.

The library does not compete with harnesses on planning loops, sandboxing, or
model quality. It competes on making the definition durable and the
delegation trustworthy. The other RFCs take these three facts as given; they
do not re-argue them.

## The three rings

**Ring 1 — the definition.** The portable artifact, in five parts:

| Part | Role | RFC |
| --- | --- | --- |
| Instruction graph | What the agent sees and which capabilities exist | 0002 |
| Tools | The typed, transactional side-effect boundary | 0003 |
| Policies | Invariants that gate actions, fail closed | 0004 |
| Feedback & completion | Trajectory steering and termination gates | 0005 |
| Event ledger | One high-granularity stream; state and transcript as views | 0006 |

The definition MUST be constructible, renderable, and testable without a
harness and without a model call. A definition that needs a live runtime to
test has already failed this RFC.

**Ring 2 — the control plane.** The contracts that make unattended operation
safe, wrapped around any harness:

| Concern | Contract | RFC |
| --- | --- | --- |
| Environment | Declared intent, mediated effects, default-deny egress | 0007 |
| Dependencies | Explicit capabilities, scoped lifecycles, injected time | 0008 |
| Bounds | Deadlines, budgets, leases tied to proof of work | 0009 |
| Distribution | At-least-once delivery, dead letters, graceful shutdown | 0010 |
| Explanation | Self-contained, queryable run records | 0011 |
| Portability | Adapter obligations and certification | 0012 |
| Iteration | Hash-anchored overrides and experiments | 0013 |
| Improvement | Evaluation as the change-control loop | 0014 |

A harness may *implement* some of these — its own sandbox, its own retries.
The control plane makes the guarantee independent of who provides it: the
definition declares what must hold, the adapter maps it, and the
compatibility suite proves it still holds.

**Ring 3 — the harness.** Model execution, the planning loop, native tools,
process isolation, workspace provisioning, transport. Constrained only
through the adapter contract (RFC-0012); a conforming library MUST NOT
require harness-side changes beyond the harness's public integration surface.

## Ownership

| Definition owns | Harness owns |
| --- | --- |
| Instruction structure and rendering | Planning loop and model calls |
| Tool schemas, semantics, permissions | Tool dispatch transport and streaming |
| Policies and their explanations | Sandbox technology and process isolation |
| Feedback triggers and completion gates | Workspace provisioning and teardown |
| Event schemas, state views, snapshots | Native tools and native observability |
| Transactional semantics of effects | Infrastructure scheduling and recovery |
| Failure classification, run-record content | Run-record transport and storage |

The adapter maps between the columns and may not move anything from left to
right. Note that gating is not owning: the harness owns its native tools; the
definition's policies still gate them (RFC-0004).

## Posture

Three postures recur throughout the collection and are normative:

**Make the correct path the easy path.** Shape incentives before enforcing
constraints: instructions co-located with the tools they document, typed
contracts that make invalid payloads unrepresentable, progressive disclosure
that keeps attention on what matters now, explicit state the model can reason
over. Gates are the backstop, not the strategy.

**Fail closed and explain.** When uncertain — a policy cannot decide,
completion cannot be verified, an environment cannot be fully materialized —
deny, block, or tear down, with a reason the agent can act on and an operator
can audit. Silent degradation is the enemy of unattended operation.

**Portability is proven, not asserted.** Every contract is phrased as
observable behavior so a certification suite can test it on any adapter.
"Same definition on a different harness" is a test result, not a claim.

## Non-goals

- **Orchestration graphs.** Where sequence truly is invariant, a policy pins
  it; everywhere else, graphs suppress the reasoning that justified an agent.
- **Model-layer competition.** No provider-specific reasoning tricks in the
  definition surface.
- **Memory and retrieval infrastructure.** Definitions consume context;
  knowledge stores are another product.
- **Mid-run human approval.** The design target is autonomy inside controlled
  bounds; humans review definitions and run records, not individual actions.

## The honest worst cases

For adopters: the harness market consolidates and portability is never
exercised. They still keep one reviewable definition, explicit policies,
transactional tools, completion gates, deterministic tests, and complete run
records. Portability is the headline benefit, not the only one.

For the abstraction: adapter drift — adapters quietly reinterpreting the
contracts until portability is a belief rather than a property. This is why
the compatibility suite (RFC-0012) is load-bearing, and why every RFC ends by
naming its observable surface.
