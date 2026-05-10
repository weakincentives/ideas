# Principles

These are design constraints, not slogans. When designs conflict, prefer the
one that better preserves the model-harness stack, caller-owned work identity,
remote sandbox correctness, and evidence about what happened.

## 1. Treat Model and Harness as One Stack

The stable target is not a bare model API. Modern agentic models are trained,
post-trained, evaluated, and shipped with a harness: tool protocols, edit
loops, filesystem shape, approval modes, message formats, and recovery
behavior.

## 2. Do Not Rebuild the Harness

Planning loops, built-in tools, provider integration, scheduling, sandboxing,
and runtime recovery belong to the harness. A definition library drives that
stack, integrates with it, and makes it observable.

## 3. Own the Definition and Integration Layers

Application teams own agent definitions: prompt structure, declared tools,
policies, feedback, completion checks, state reducers, resources, output
contracts, and evaluation fixtures.

They also own the integration layer: harness adapters, tool bridges, skills,
workspace integration, evals, contract tests, and operational evidence. That is
where the long-term leverage lives.

## 4. The Prompt Is Executable Structure

The prompt is not just text. It is a typed, hierarchical document that carries
instructions, capabilities, visibility rules, output contracts, and local
context. Rendering is deterministic, inspectable, and reproducible.

No hidden registry may disagree with the prompt about what the agent can do.

## 5. Co-Locate Capability and Instruction

The section that explains a capability declares it. The section that describes
an output format declares the output type. Structure makes drift harder.

## 6. Put Operating Knowledge in Skills

Skills package the operating knowledge a harness needs to use a repository,
tool, workflow, or domain correctly. They are not decorative prompt fragments.
They are versioned integration assets that travel with the harness adapter and
are tested against the harness.

## 7. Expose Side Effects as Tools

Application access to outside systems flows through declared tools, not broad
network access from inside the sandbox. Every application action has a name,
typed input, typed output, policy boundary, idempotency story, and trace.

## 8. Use Policies for Rules

Encode rules, not brittle procedures. Policies fail closed and produce typed
denial records. Feedback can guide behavior, but it does not authorize side
effects.

## 9. Keep State Event-Driven

State changes reduce from typed events. Durable state is serializable,
inspectable, and replayable. Runtime implementation details are not required to
understand what happened.

## 10. Scope Transaction Claims

Transactions cover only framework-owned state and snapshotable resources. They
do not roll back completed external API calls, unwrapped built-in harness
commands, or filesystem mutations outside the declared snapshot boundary.

## 11. Use Typed Contracts at Boundaries

Parameters, tool calls, tool results, policies, events, state slices,
structured output, protocol messages, and debug records are named typed values.
The host language chooses the representation; the discipline is the same.

## 12. Design Remote First

Assume the harness and filesystem run in a separate sandbox reached through a
protocol. Shared memory, shared file descriptors, in-process callbacks, and
local paths are development conveniences, not design premises.

## 13. Work Identity Belongs to the Caller

The caller supplies stable identifiers for durable work. Backend session IDs,
provider request handles, and runtime trace tokens are private integration
state. Public protocol messages use caller-owned names.

## 14. Transport Is Not Ownership

A connection is not work. Disconnecting does not cancel work by default.
Reconnecting does not start new work. Compute lifecycle, work lifecycle, and
connection lifecycle remain separate.

## 15. The Workspace Is the Data Plane

The agent workspace is not a temporary directory on the caller's machine. In
production it is a remote, tenant-scoped, persistent filesystem mounted near
the harness. Compute can restart without deleting it.

## 16. Keep Evidence for Every Run

Every run emits enough typed evidence to reconstruct what happened: rendered
definition, tool schemas, tool calls, built-in harness events, filesystem
snapshots or references, outputs, budgets, errors, and trace correlation.

## 17. Contract Support Is Proved, Not Assumed

An integration layer passes shared contract tests before claiming portability.
The tests cover rendering, tool bridging, harness adapters, skills, policies,
state, transactions, durable work, reconnect, evidence, and structured
output.
