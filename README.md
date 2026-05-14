# The Agent-Definition Layer

A six-pager on a portable agent-definition layer: what the pattern is, why it
matters, and what any implementation of this shape must preserve.

This document is not a product specification for one framework. It describes the
implementation contract for any framework that separates durable agent
definitions from execution harnesses. Specific project names can change. The
boundary is the important part.

______________________________________________________________________

## 1. Thesis

Every production agent has two halves: the **definition** and the **harness**.

The definition is what the agent *is*: its instruction structure, its tools, its
policies, its state contract, its completion criteria, and the feedback it uses
to decide whether work is still unfinished. The harness is what the runtime
*does*: model calls, planning loops, sandboxing, tool dispatch, retries,
scheduling, deadline enforcement, workspace provisioning, and crash recovery.

For the first generation of agent software, these halves were usually built by
the same team in the same repository. That was tolerable while the harness was a
thin loop around a model endpoint. It is no longer the right default. Harnesses
are becoming serious runtime products. They differ in planning strategy,
permission model, sandbox fidelity, native tools, workspace lifecycle,
observability, scheduling, and recovery semantics. They are also improving too
quickly for most application teams to treat one harness as the permanent center
of their agent architecture.

At the same time, the workspace an agent operates on is becoming less likely to
be a developer's local checkout. Increasingly it is an ephemeral, sandboxed, or
remote workspace provisioned by the harness. Sometimes it is near the model.
Sometimes it is in a container. Sometimes it is in a customer-controlled
environment. The definition should not need to know. A tool that reads, writes,
or validates workspace state should have the same preconditions, effects,
rollback behavior, and observable trace whether the backing workspace is local,
containerized, remote, or virtualized.

The agent-definition layer exists because the definition should not be locked to
one harness and should not be entangled with how that harness is implemented. It
is a portable, reviewable, typed artifact that specifies what the agent is. Thin
adapters carry that artifact to whichever harness is appropriate for a given
task, organization, model, security boundary, or deployment environment.

The bet is simple: **the harness is the depreciating asset; the definition is
the durable one**. Teams that fuse the two will rewrite their agents every time
a better harness appears. Teams that separate them can preserve the reasoning
structure, tool surface, policies, and completion criteria that encode their
actual agent IP.

______________________________________________________________________

## 2. Tenets

1. **The definition is the source of truth.** Instructions, tools, policies,
   feedback, and completion criteria live in one coherent definition artifact.
   There is no separate registry that can drift from the rendered instructions
   and no hidden configuration that changes the agent's capabilities outside
   review.

1. **Own the definition; rent the harness.** Planning loops, sandboxes, retries,
   leases, process isolation, scheduling, and recovery are runtime concerns.
   They are important, but they are not the durable application asset. The
   durable asset is the agent's reasoning structure, allowed actions,
   invariants, and success criteria.

1. **Policies over workflows.** Encode the invariants the agent must satisfy,
   not the steps it must follow. Workflows are useful when sequence is genuinely
   invariant. For uncertain work, procedural orchestration often suppresses the
   reasoning capability that justified using an agent in the first place.
   Policies constrain the search space without pretending the author knows the
   path in advance.

1. **Tools are transactional side-effect boundaries.** Every externally visible
   tool call has typed input, typed output, explicit permissions, and atomic
   commit semantics. The framework snapshots relevant state before the call and
   rolls back on failure. Failed tools should not leave partial traces that make
   retries unsafe or recovery ambiguous.

1. **State is an event ledger.** Mutations flow through typed events and pure
   reducers. Snapshots are first-class. State is inspectable, replayable where
   the underlying evidence still exists, and explainable after the run. There is
   no hidden mutable state that must be reconstructed from ad hoc logs.

1. **Fail closed; surface reasons; test compatibility.** When a policy is
   uncertain, deny. When completion is uncertain, block termination. Emit a
   reason the agent can read and an operator can audit. Adapter compatibility is
   something a framework proves with tests, not something it asserts in
   documentation.

______________________________________________________________________

## 3. The Situation

Early agent frameworks were chat loops with tool plumbing. A team wrote a
prompt template, registered a few tools, parsed tool calls, retried failures,
and kept enough logs to debug obvious failures. The boundary between "agent" and
"runtime" was blurry because the runtime did very little.

That world is being replaced by full agent harnesses. A modern harness may own
the planning loop, tool sequencing, filesystem access, process isolation,
network policy, lease extension, long-running task recovery, multi-agent
coordination, and native observability. Different harnesses make different
tradeoffs. Some are optimized for local developer workflows. Some are optimized
for unattended remote execution. Some are tied to a model provider. Some are
vendor-neutral protocols. The set will continue changing.

The workspace model is changing with it. An unattended agent may no longer be a
process with direct access to the operator's machine. It may operate inside a
short-lived sandbox, a remote container, a virtual filesystem, or an environment
with strict network and data-access controls. The harness may be responsible for
provisioning the workspace, mounting inputs, enforcing permissions, capturing
artifacts, and tearing everything down.

Teams building serious agents now sit between those shifts. They want to use the
best available harness for sandboxing, native capabilities, recovery, cost, and
model quality. They also want agent behavior that is stable enough to review,
test, audit, and improve over time. Without a definition layer, those goals
conflict. Every harness upgrade becomes a port. Every workspace change exposes
local filesystem assumptions. Every provider-specific feature creeps into the
agent's core behavior.

This friction creates accidental lock-in. Teams stay on a harness not because it
is still the right one, but because the thing they call "the agent" is actually
a fused bundle of prompts, tool handlers, workflow code, sandbox assumptions,
state mutations, and runtime-specific error handling.

______________________________________________________________________

## 4. The Problem We Are Solving

There is no widely accepted artifact that represents "the agent" independently
from "the agent running on this harness." In many systems, describing the agent
means describing a tangle: instructions in one file, a tool registry assembled
elsewhere, permissions enforced by runtime glue, completion checks bolted on
after the fact, state spread across mutable objects, and implicit assumptions
about where files live.

That tangle is not portable, not reviewable as a unit, not testable in
isolation, and not robust to harness churn.

The failure modes are concrete:

- **Accidental lock-in.** A team adopts a harness for a good reason and later
  discovers that the agent cannot move because the definition and runtime are
  fused at every layer.
- **Workflow brittleness.** Agents encoded as procedural workflows either fail
  when reality diverges from the path or accumulate decision trees that are
  harder to audit than the work they replaced.
- **Premature termination.** Unattended agents declare success while work
  remains unfinished. Without definition-level completion criteria, there is no
  portable way to block stop or explain what is missing.
- **Untestable behavior.** Regression testing requires standing up the full
  harness because there is no smaller artifact that captures the agent's
  policies, tool contract, and completion logic.
- **Filesystem coupling.** Tools written against local assumptions must be
  rewritten for remote, sandboxed, or virtual workspaces. Path semantics,
  permissions, streaming behavior, snapshots, and rollback all drift.
- **Operational opacity.** Without a unified event model, a run record can
  become a pile of logs rather than an explanation of what happened and why.
- **Adapter drift.** If each runtime bridge gets to reinterpret behavior, "same
  definition on a different harness" decays into "similar behavior if nothing
  unusual happens."

The root cause is the same in each case: the system lacks a first-class
definition layer with a clear compatibility contract. Everything called "the
agent" is really "the agent plus one runtime implementation, fused."

______________________________________________________________________

## 5. Reference Architecture

An implementation of this pattern should treat the agent definition as a
portable artifact with five cooperating parts: instruction tree, tools,
policies, feedback, and state contract. The harness remains outside the
definition. The adapter is the narrow bridge between them.

**The instruction tree** is the agent's reasoning surface. It is hierarchical
and deterministic. Sections can contain instructions, child sections,
enablement conditions, attached tools, local policies, and completion signals.
Rendering the tree produces the instructions the agent sees. Enabling or
disabling a section changes both instructions and capabilities together, so
documentation and tool availability cannot drift apart. The rendered definition
is reviewable as the effective contract for what the agent can see and do.

**The tools** are the side-effect boundary. They are typed at input and output,
declared in the definition, and executed through narrow resource protocols
rather than direct assumptions about the host process. A workspace protocol
abstracts file reads, writes, listing, streaming, snapshots, and restore. A
clock can be injected. Secrets and external resources are resolved through
explicit capabilities with lifetimes and permissions. Tool handlers should not
know which harness invoked them or where the workspace physically lives.

**The policies** are declarative invariants that gate action. A policy might say
a file must be read before overwrite, tests must pass before deployment, a tool
requires a particular capability, or a long operation must heartbeat before its
lease expires. Policies fail closed and return explanations. They do not script
the agent's path. They define what must hold for the next action to be allowed.

**The feedback** is the definition's answer to "is the agent still on track?"
Feedback providers observe the trajectory and add advisory context at useful
moments. Completion checkers block termination when the definition's own success
criteria are not satisfied. Both belong to the definition because both describe
the work the agent is supposed to perform, not the mechanics of the harness
running it.

**The state contract** records behavior as typed events reduced into state.
State includes tool calls, policy decisions, resource snapshots, transcript
events, feedback injections, completion decisions, and emitted artifacts. The
event ledger should be good enough to answer why the agent acted, what evidence
it had, what policies constrained it, and which effects committed. Full replay
may be impossible when external data expires or privacy boundaries prevent
retention, but the run record should still preserve intent, evidence references,
integration contracts, and decision points.

These parts constitute the portable definition. They are version-controlled,
reviewable in one change, and testable without a harness. Unit tests should be
able to validate policy decisions, tool schemas, completion checks, reducers,
rendering, and transactional rollback without making a model call.

The **adapter** is intentionally thin. It maps the rendered definition, tool
surface, policy gates, feedback hooks, and event stream onto a particular
harness protocol. The adapter may translate transport, authentication,
registration, streaming, and lifecycle events. It may not redefine tool
semantics, weaken policies, bypass completion checks, or mutate state outside
the event contract.

Every serious implementation of this architecture needs a compatibility suite.
The suite should certify that an adapter preserves the same observable semantics
as the reference behavior:

- the same rendered instruction surface for the same enabled sections
- the same tool schemas and permission gates
- transactional commit and rollback on tool success and failure
- fail-closed policy behavior with surfaced explanations
- completion checks that block termination consistently
- structurally equivalent event ledgers and run records
- workspace behavior that is independent of local, sandboxed, or remote backing
  storage

Without that suite, portability is a claim. With it, portability becomes an
engineering property that can fail tests.

______________________________________________________________________

## 6. What Success Looks Like

A team writes an unattended agent once. The instruction tree describes the work.
The tools expose a small typed side-effect surface. The policies state the
invariants. The completion checker encodes the deliverable. The state contract
records what happened. The team tests those pieces without launching any
harness.

They run the same definition on a local harness for fast iteration. Later they
run it on a remote sandbox for unattended execution. Later still they compare a
new harness because it has better isolation, scheduling, cost, model access, or
observability. The definition does not change. Only the adapter and runtime
configuration change.

A new harness appears. An adapter is written. It passes the compatibility suite.
Existing definitions become runnable there without rewriting prompts, tools, or
policies. Harness improvements compound across the fleet because agent behavior
was not trapped inside one runtime.

A new engineer reads one definition to understand what the agent is allowed to
do and what counts as done. An auditor asks what the agent can touch, what it
must prove before acting, and why it stopped. The answer is not "read the
runtime code and reconstruct a log stream." The answer is the definition and its
run record.

That is the success state. The center of gravity in agent engineering moves
from runtime plumbing to definition design: instruction structure, tool surface,
policy set, feedback loop, state contract, and completion criteria. The harness
still matters, but it is no longer where the agent's durable identity lives.

______________________________________________________________________

## 7. Frequently Asked Questions

**Is this trying to replace agent harnesses?**

No. The harnesses solve hard runtime problems: model execution, sandboxing,
tool dispatch, process isolation, retries, deadlines, recovery, scheduling, and
workspace lifecycle. The definition layer depends on harnesses. It only insists
that the definition of the agent not be fused to one of them.

**Why not standardize on one harness and accept the lock-in?**

Sometimes that is a reasonable short-term product choice. It should not become
an architectural accident. Harnesses are still changing quickly, model access
changes, security requirements differ by customer, and remote execution
environments vary. A durable definition gives a team the option to move when the
runtime tradeoffs change.

**Why policies over workflows? Are workflows wrong?**

Workflows are right when sequence is genuinely invariant: protocol handshakes,
regulated procedures, deterministic pipelines, or small automations where the
path is the product. For open-ended agent work, the useful abstraction is often
the invariant, not the path. Policies preserve the agent's ability to reason
while still making the action boundary explicit and auditable.

**What does a source-of-truth definition buy?**

It buys co-location, reviewability, and testability. The instructions for using
a tool live with the tool declaration. The policies and completion checks that
define acceptable behavior live with the instructions. A review of the
definition is a review of the agent's effective behavior, rather than a scavenger
hunt across prompts, registries, runtime glue, and logs.

**How does this hold up when the workspace is remote?**

Tools talk to a workspace protocol, not directly to local process assumptions.
The protocol covers reads, writes, listing, streaming, snapshots, restore, and
permissions. Backends can implement the same protocol for local directories,
containers, remote sandboxes, virtual filesystems, or provider-managed
workspaces. The definition observes the same semantics either way.

**Where does the definition end and the harness begin?**

The definition owns instructions, tools, policies, feedback, completion checks,
state reducers, event schemas, transactional semantics, and the structured
run-record contract. The harness owns model execution, planning mechanics,
process isolation, scheduling, retries, deadlines, workspace provisioning,
transport, and native runtime capabilities. The adapter maps between the two
without changing the definition's behavioral contract.

**What happens when a new harness ships?**

An adapter is written for its protocol. The adapter runs against the
compatibility suite. If it preserves the definition's semantics, existing
definitions can run on the new harness. If it cannot preserve those semantics,
the mismatch is explicit rather than hidden in runtime glue.

**Is this overengineered for current agent workloads?**

For short foreground assistants, often yes. For unattended agents that operate
for long periods, touch real resources, run in remote workspaces, and need audit
trails, the structure is not excess ceremony. It is the minimum needed to keep
behavior reviewable, recoverable, and portable.

**What is the worst case for adopters?**

The harness market consolidates and portability becomes less valuable than
expected. Even then, adopters retain a single reviewable definition, explicit
policies, transactional tool semantics, completion gates, testable behavior, and
better run records. Portability is the headline benefit, not the only one.

**What is the worst case for the abstraction?**

Adapter drift. If adapters reinterpret tool semantics, weaken policies, skip
completion gates, or emit incompatible event records, the definition is no
longer portable in practice. The compatibility suite is therefore load-bearing.
It is what keeps the abstraction honest.

______________________________________________________________________

## 8. Appendix: The Boundary Diagram

```text
+------------------------------------------------------------------+
| DEFINITION                                                       |
|                                                                  |
| Authored by the team. Versioned with the product. Portable.      |
|                                                                  |
| - Instruction tree: deterministic, hierarchical, scoped          |
| - Tools: typed I/O, explicit permissions, transactional effects  |
| - Policies: declarative invariants, fail-closed explanations     |
| - Feedback: trajectory guidance and completion gates             |
| - State contract: typed events, reducers, snapshots, run records |
|                                                                  |
| Reviewable and testable without a harness.                       |
+------------------------------+-----------------------------------+
                               |
                               | Adapter
                               | Thin translation layer.
                               | Certified by a compatibility suite.
                               v
+------------------------------------------------------------------+
| HARNESS                                                          |
|                                                                  |
| Owned by the runtime provider or platform team. Swappable.       |
|                                                                  |
| - Model execution and planning loop                              |
| - Tool dispatch transport and streaming                          |
| - Process isolation, sandboxing, permissions, network policy     |
| - Workspace provisioning and lifecycle                           |
| - Scheduling, retries, deadlines, lease extension, recovery      |
| - Native observability and orchestration                         |
|                                                                  |
| Improves independently from the definition.                       |
+------------------------------------------------------------------+
```

The line between the boxes is the architectural commitment. Everything else is
implementation detail.
