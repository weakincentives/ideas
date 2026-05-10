# Principles

These are the rules behind the rest of the repository. Prefer designs that make
remote work safe, keep the model-harness stack intact, and leave enough records
to understand what happened.

## 1. Treat Model and Harness as One Stack

The stable target is not a bare model API. Modern agentic models are trained,
evaluated, and shipped with a harness: tool protocols, edit loops, filesystem
shape, approval modes, message formats, and recovery behavior.

## 2. Do Not Rebuild the Harness

The harness already owns the planning loop, built-in tools, provider
integration, scheduling, sandboxing, and runtime recovery. A definition library
drives that stack, connects it to application systems, and records what it does.

## 3. Own Definitions and Integration

Application teams own the agent definition: prompt structure, declared tools,
policies, feedback, completion checks, state reducers, resources, outputs, and
eval fixtures.

They also own the integration layer: harness adapters, tool bridges, skills,
workspace integration, evals, contract tests, and run records. That is where the
durable work lives.

## 4. Keep Instructions Near Capabilities

The prompt is not just text. It is structured input that tells the agent what it
can see, what it can do, which rules apply, and what output is expected.

A tool appears near the instructions that teach the agent how to use it. A
policy appears near the behavior it constrains. An output format appears near
the output it validates. Keeping these together makes drift harder.

## 5. Put Operating Knowledge in Skills

Skills package the operating knowledge a harness needs to use a repository,
tool, workflow, or domain correctly. They are not decorative prompt fragments.
They are versioned integration assets that travel with the harness adapter and
are tested against the harness.

## 6. Put Side Effects Behind Tools

Application access to outside systems goes through declared tools, not broad
network access from inside the sandbox. Every application action has a name,
typed input, typed output, policy check, idempotency story, and trace.

## 7. Use Policies for Rules

Policies encode rules and fail closed. If a policy cannot evaluate a request, it
denies or stops the action and records why. Feedback can guide behavior, but it
does not authorize side effects.

## 8. Keep State Explainable

State changes come from typed events. Durable state is serializable,
inspectable, and replayable. A reader should not need to know runtime internals
to understand what happened.

## 9. Be Honest About Rollback

Transactions only cover state the definition library controls. They do not roll
back an external API call that already happened, an unwrapped built-in harness
command, or a filesystem mutation outside a sandbox snapshot.

## 10. Design for Remote Sandboxes First

Assume the harness and filesystem run in a separate sandbox reached through a
protocol. Shared memory, in-process callbacks, and local paths are development
conveniences, not design premises.

## 11. Let the Caller Name Work

The caller supplies stable names for durable work. Backend session IDs, provider
request handles, and runtime trace tokens can exist, but public protocol
messages use caller-owned names.

## 12. Keep Work, Compute, and Connections Separate

A connection is not work. Disconnecting does not cancel work by default.
Reconnecting does not start new work. A sandbox process can restart without
deleting the workspace or the caller's work record.

## 13. Treat the Workspace as Durable Data

The agent workspace is not a temporary directory on the caller's machine. In
production it is a remote, tenant-scoped, persistent filesystem mounted near the
harness.

## 14. Keep Evidence for Every Run

Every run leaves enough evidence to reconstruct what happened: rendered
definition, tool schemas, tool calls, built-in harness events, filesystem
snapshots or references, outputs, budgets, errors, and trace correlation.

## 15. Prove the Shared Contract

An integration layer passes shared contract tests before claiming cross-harness
support. The tests cover rendering, tool bridging, harness adapters, skills,
policies, state, transactions, durable work, reconnect, run records, and
structured output.
