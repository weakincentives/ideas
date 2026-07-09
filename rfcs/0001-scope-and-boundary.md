# RFC-0001: Scope and the Definition/Harness Boundary

- Status: Draft
- Ring: framing
- Depends on: none

## Summary

A library for unattended agents should be built around one architectural
commitment: the **agent definition** is a first-class, portable artifact that
is authored, versioned, reviewed, and tested independently of the **execution
harness** that runs it. Between them sits a deliberately thin, certified
**adapter**. Around them sits a **control plane**: the contracts that make it
safe to hand a definition to a harness and walk away — bounded execution,
durable work distribution, a complete run record, versioned iteration, and
evaluation.

This RFC fixes the vocabulary, the ownership boundaries, and the design
posture that every other RFC in the collection assumes.

## Motivation

Unattended agents differ from foreground assistants in three ways that
dominate the design:

1. **Nobody is watching.** There is no human in the loop to notice drift,
   premature termination, runaway spending, or a stuck process. Every
   correction mechanism must be encoded in advance and enforced mechanically.
2. **The runtime is not yours.** Harnesses — vendor runtimes, protocol-based
   agents, local loops — are improving faster than application teams can
   track. Any coupling between the agent's identity and one harness becomes
   lock-in within quarters, not years.
3. **The workspace is not local.** The environment an agent acts on is
   increasingly ephemeral, sandboxed, remote, or virtual. Assumptions about
   local filesystems, ambient credentials, or unrestricted network access do
   not survive contact with production deployment.

The consequence is a division of labor. The library described by these RFCs
does not compete with harnesses on planning loops, sandboxing technology, or
model quality. It competes on making the *definition* durable and the
*delegation* trustworthy.

## The three rings

### Ring 1: the definition

The definition specifies what the agent **is**. It has five cooperating parts,
each with its own RFC:

| Part | Role | RFC |
| --- | --- | --- |
| Instruction graph | What the agent sees and which capabilities exist | 0002 |
| Tools | The typed, transactional side-effect boundary | 0003 |
| Policies | Invariants that gate actions, fail closed | 0004 |
| Feedback & completion | Trajectory steering and termination gates | 0005 |
| State contract | Typed events reduced into inspectable state | 0006 |

The definition MUST be constructible, renderable, and testable without a
harness and without a model call. Unit tests over policy decisions, tool
schemas, completion checks, reducers, and rendering are the cheapest and most
reliable regression net an agent team has; a definition that requires a live
runtime to test has already failed this RFC.

### Ring 2: the control plane

The control plane is everything the library must guarantee so a definition can
run **unattended**. It is not the harness — it is the set of contracts wrapped
around any harness:

| Concern | Contract | RFC |
| --- | --- | --- |
| Environment | Declared intent, mediated effects, default-deny egress | 0007 |
| Dependencies | Explicit capabilities, scoped lifecycles, injected time | 0008 |
| Bounds | Deadlines, budgets, leases tied to proof of work, watchdogs | 0009 |
| Distribution | At-least-once delivery, dead letters, graceful shutdown | 0010 |
| Explanation | Canonical transcript, self-contained run record | 0011 |
| Portability | Adapter obligations and certification | 0012 |
| Iteration | Hash-anchored overrides and experiments | 0013 |
| Improvement | Evaluation as the change-control loop | 0014 |

Some control-plane responsibilities may be *implemented* by a harness (e.g., a
vendor runtime enforces its own sandbox). The control plane's job is to make
the guarantee independent of which harness happens to provide it: the
definition declares what must hold, the adapter maps it, and the compatibility
suite proves it still holds.

### Ring 3: the harness

The harness owns model execution, the planning/act loop, tool dispatch
transport, process isolation, workspace provisioning, scheduling, retries at
the infrastructure level, and crash recovery. These RFCs constrain the harness
only through the adapter contract (RFC-0012). A conforming library MUST NOT
require harness-side changes beyond what its adapter can express through the
harness's public integration surface.

## Design posture

Three postures recur throughout the collection and are normative:

**Make the correct path the easy path.** The library shapes incentives rather
than scripting behavior. Instructions co-located with the tools they document,
typed contracts that make invalid payloads unrepresentable, progressive
disclosure that keeps attention on what matters now, and explicit state that
gives the model real context — these make the model's easiest continuation the
correct one. Constraint mechanisms (policies, gates) exist, but they are the
backstop, not the strategy.

**Fail closed and explain.** Whenever the library is uncertain — a policy
cannot decide, completion cannot be verified, an environment cannot be fully
materialized — it MUST deny, block, or tear down, and it MUST surface a reason
that both the agent can reason about and an operator can audit. Silent
degradation is the enemy of unattended operation.

**Portability is proven, not asserted.** Every behavioral contract in these
RFCs is phrased as an observable requirement so a certification suite can test
it against any adapter. "Same definition on a different harness" is an
engineering property with failing tests, not a documentation claim.

## Ownership boundary

The single most load-bearing table in the collection:

| Definition owns | Harness owns |
| --- | --- |
| Instruction structure and rendering | Planning loop and model calls |
| Tool schemas, semantics, permissions | Tool dispatch transport and streaming |
| Policies and their explanations | Sandbox technology and process isolation |
| Feedback triggers and completion gates | Workspace provisioning and teardown |
| Event schemas, reducers, snapshots | Scheduling, infrastructure retries, recovery |
| Transactional semantics of effects | Native tools and native observability |
| Run-record content contract | Run-record transport/storage mechanics |

The adapter maps between the columns. It MUST NOT move an item from the left
column to the right: an adapter that reinterprets tool semantics, weakens a
policy, skips a completion gate, or emits a nonconforming event stream is
nonconforming, however convenient the shortcut.

## Non-goals

- **Graph-based orchestration.** The library does not provide node/edge
  workflow composers. Where sequence truly is invariant, a policy can encode
  the dependency (RFC-0004); where it is not, orchestration graphs suppress
  the reasoning that justified using an agent.
- **Model-layer competition.** No prompt-optimization tricks tied to one
  provider, no provider-specific reasoning APIs in the definition surface.
- **Memory and retrieval infrastructure.** Definitions consume context;
  building knowledge stores is another product's job.
- **Human-in-the-loop ceremony.** The design target is agents that operate
  autonomously inside controlled environments. Human review happens at the
  definition and run-record level, not as mid-run approval gates.

## The worst cases, stated honestly

For adopters: the harness market consolidates and portability is never
exercised. Even then, the team keeps a single reviewable definition, explicit
policies, transactional tools, completion gates, deterministic tests, and
complete run records. Portability is the headline benefit, not the only one.

For the abstraction: adapter drift. If adapters quietly reinterpret the
contracts, the definition stops being portable in practice while everyone
still believes it is. This is why RFC-0012's compatibility suite is
load-bearing and why every RFC ends by naming its observable surface.
