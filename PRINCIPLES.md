# Principles

These are the rules behind the rest of the repository. Prefer designs that keep
the model-harness stack intact, keep the definition in the middle, make remote
work safe, and leave enough records to understand what happened.

## 1. Treat Model and Harness as One Stack

For the systems this repository targets, the stable target is not a bare model
API. The useful behavior comes from a model used through a harness: tool
protocols, edit loops, filesystem shape, approval modes, message formats, and
recovery behavior.

## 2. Do Not Rebuild the Harness

The harness already owns the planning loop, built-in tools, provider
integration, scheduling, sandboxing, and runtime recovery. A definition library
drives that stack, connects it to application systems, and records what it does.

## 3. Keep the Definition in the Middle

The definition is the contract between application intent and runtime behavior.
It is not the application, and it is not the harness.

Application-facing integration binds product data, tools, policies, resources,
outputs, eval fixtures, and caller-owned work names into the definition.

Runtime-facing integration renders that definition into one harness, stages
skills, connects workspaces, routes tool calls, streams events, and handles
runtime differences through harness adapters and sandbox protocols.

## 4. Keep Instructions Near Capabilities

The prompt is structured input, not just text. It tells the agent what it can
see, what it can do, which rules apply, and what output is expected.

A tool appears near the instructions that teach the agent how to use it. A
policy appears near the behavior it constrains. An output format appears near
the output it validates.

## 5. Use Tools for Side Effects and Skills for Operating Knowledge

Application side effects go through declared tools, not broad network access
from inside the sandbox. Every application action has a name, typed input,
typed output, policy check, idempotency story, and trace.

Skills package operating knowledge: how to use a repository, workflow, domain,
or tool correctly. Skills do not replace tools, and tools do not replace skills.

## 6. Design for Remote Sandboxes First

Assume the harness and filesystem run in a separate sandbox reached through a
protocol. Shared memory, in-process callbacks, and local paths are development
conveniences, not design premises.

## 7. Let the Caller Name Work

The caller supplies stable names for durable work. Backend session IDs, provider
request handles, and runtime trace tokens can exist, but public protocol
messages use caller-owned names.

## 8. Keep Work, Compute, and Connections Separate

A connection is not work. Disconnecting does not cancel work by default.
Reconnecting does not start new work. A sandbox process can restart without
deleting the workspace or the caller's work record.

## 9. Be Honest About Rollback

Transactions only cover state the definition library controls. They do not roll
back an external API call that already happened, an unwrapped built-in harness
command, or a filesystem mutation outside a sandbox snapshot.

## 10. Leave Records and Prove Behavior

Every run leaves enough evidence to reconstruct what happened: rendered
definition, tool schemas, tool calls, built-in harness events, filesystem
snapshots or references, outputs, budgets, errors, and trace correlation.

An integration layer passes shared contract tests before claiming cross-harness
support.
