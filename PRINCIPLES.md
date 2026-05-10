# Principles

These are operative constraints, not slogans. When two designs conflict, prefer
the one that better respects model-harness coupling, preserves durable work
identity, and improves remote sandbox correctness.

## 1. Model and Harness Are One Substrate

The model is not independent from the harness that surrounds it. Modern agentic
models are trained, post-trained, evaluated, and shipped in the context of
specific tool protocols, edit loops, filesystem assumptions, approval modes,
message formats, and recovery behavior.

Treat the model-harness pair as the execution substrate. Do not design as if a
bare model API is the stable target.

## 2. Do Not Rebuild the Harness

The planning loop, native tools, provider integration, scheduling, sandboxing,
and runtime recovery are harness responsibilities. A definition library should
not compete with that stack. Its job is to drive it, integrate with it, and
make its behavior observable.

## 3. Own Definitions, Drivers, Integrations, and Skills

The agent definition is the artifact users own. It includes prompt structure,
declared tools, policies, feedback providers, completion checks, state
reducers, resources, output contracts, and evaluation fixtures.

Teams should also own the drivers, tool bridges, skills, workspace integration,
evals, and observability surfaces that connect their systems to the rented
execution substrate. That is where durable leverage lives.

## 4. The Prompt Is Executable Structure

The prompt is not just text. It is a typed, hierarchical document that carries
instructions, capabilities, visibility rules, output contracts, and local
context. Rendering should be deterministic and inspectable.

There should not be a second hidden registry that disagrees with the prompt
about what the agent can do.

## 5. Co-Locate Instruction and Capability

The section that explains a capability should be the section that declares it.
The section that describes an output format should declare the output type. This
makes drift structurally harder.

## 6. Skills Are Integration Assets

Skills package the operational knowledge a harness needs to use a repository,
tool, workflow, or domain correctly. They are not decorative prompt fragments.
They are versioned integration assets that should travel with the driver and be
tested against the harness.

## 7. Policies, Not Workflows

Encode invariants, not brittle procedures. A policy says what must hold. A
workflow says what to do next. Use workflows only for sequences that are truly
protocol-level and invariant.

## 8. Fail Closed

If a constraint cannot be evaluated, deny or stop by default. The denial should
be typed and observable so the agent or caller can recover deliberately.

## 9. Tools Are the Application Egress Surface

Application-level external access should flow through declared tools, not
ambient network access from inside the sandbox. The model and harness may need
provider egress; application egress is a capability and should be named,
authorized, observed, and testable.

## 10. State Is Event-Driven

State changes should be reducible from typed events. Durable state should be
serializable, inspectable, and replayable. Runtime implementation details should
not be required to understand what happened.

## 11. Transactions Are Scoped Guarantees

Transactions are valuable, but they are not magic. A framework can roll back
session state and snapshotable resources it controls. It cannot roll back an
external API call that already happened, a native harness command it did not
wrap, or a filesystem mutation outside the declared snapshot boundary.

## 12. Typed Contracts Everywhere

Parameters, tool calls, tool results, policies, events, state slices,
structured output, protocol messages, and debug records should be named typed
values. The host language decides the representation; the discipline is the
same.

## 13. Remote by Design

Assume the harness and filesystem run in a separate sandbox reachable only
through a protocol. Shared memory, shared file descriptors, in-process
callbacks, and local paths are development conveniences, not design premises.

## 14. Work Identity Belongs to the Caller

The caller supplies stable identifiers for durable work. Backend-native session
IDs, provider request handles, and runtime trace tokens are private driver
state. Public protocol payloads should use caller-owned names.

## 15. Transport Is Not Ownership

A connection is not work. Disconnecting should not cancel work by default.
Reconnecting should not start new work. Compute lifecycle, work lifecycle, and
connection lifecycle must remain separate.

## 16. The Workspace Is Durable Data Plane

The agent workspace is not a temporary directory on the caller's machine. In
production it is a remote, tenant-scoped, persistent filesystem mounted near
the harness. Compute can restart without deleting it.

## 17. Observability Is a Contract

Every run should emit enough typed evidence to reconstruct what happened:
rendered definition, tool schemas, tool calls, native harness events,
filesystem snapshots or references, outputs, budgets, errors, and trace
correlation.

## 18. Compatibility Is Proved, Not Assumed

A driver should pass a shared compatibility suite before claiming portability.
The suite should cover rendering, tool bridging, skills, policies, state,
transactions, durable work, reconnect, observability, and structured output.
