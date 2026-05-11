# Integration Layer

The integration layer has two directions. One faces the application: it turns
product systems into inputs the agent definition can use. The other faces the
runtime: it runs the definition through a real harness, sandbox, workspace,
skill format, and event stream. The [agent definition](DEFINITION.md) sits
between them.

Naming the directions separately matters. They have different owners,
different failure modes, different tests, and different audit needs. A single
"integration" blob collapses concerns that should be kept apart.

## Why It Exists

A harness is not a neutral shell around a model. The useful behavior comes
from the model as used through the harness's tools, edit loop, files,
approvals, transcripts, skills, and recovery. The integration layer lets an
application benefit from that behavior without hard-coding the application to
one harness, and it keeps the harness from reaching directly into application
systems.

## Application-Facing Integration

Application-facing integration answers: how does product intent enter the
agent system?

It binds:

- user and tenant context
- tool handlers
- query engines and data systems
- metric definitions and data ownership
- authorization and policy checks
- resources and data sources
- output consumers
- eval fixtures and test cases
- product-level budgets and deadlines
- analytical cost and freshness limits
- review, escalation, and delivery rules
- retry and conflict behavior

This direction produces clear definition inputs. It should not depend on one
harness's private prompt format or runtime behavior. When done well, an
application team can change harnesses without rewriting the code that binds
its tools and policies.

## Runtime-Facing Integration

Runtime-facing integration answers: how does this definition run in this
harness and sandbox?

It owns:

- rendering the definition for one harness
- packaging skills in the harness's format
- staging runtime files
- translating tool calls back to application handlers
- routing query tool calls back to data-system handlers
- translating harness events into standard events
- reporting unsupported requirements instead of hiding them
- mapping public work identifiers to sandbox work
- calling the sandbox protocol for workspace and lifecycle operations
- preserving harness-specific errors and terminal results

This direction is where harness adapters live. A harness adapter translates
the shared definition into one harness's rules and formats. It does not hide
differences between harnesses; it makes them clear enough to test.

For example, a Codex adapter and a Claude Agent SDK adapter may both run the
same definition. They should not pretend their tools are identical. Each
adapter may render skills, approvals, edits, shell commands, and event
streams differently. The shared layer preserves application intent; the
adapter preserves harness behavior. See [SKILLS.md](SKILLS.md) for how
skill packages travel alongside the definition across harnesses.

## Sandbox Protocol Boundary

The integration layer calls the sandbox protocol. It does not own the
sandbox. The sandbox protocol owns runtime lifecycle, workspace access,
reconnect, isolation, external access policy, durable runtime records, and
control-plane state. See [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md) for that
boundary in detail.

This line matters. The integration layer maps application intent onto the
runtime environment. The sandbox protocol manages the runtime environment.
Crossing the line by reaching into sandbox internals from application code
produces coupling that breaks the next time a sandbox is upgraded.

## Built-In Harness Tools

Every harness has built-in tools. A harness adapter classifies each one:

- unavailable
- available but hidden from the definition
- visible only through observed events
- gated by sandbox policy
- wrapped behind a definition tool
- covered by sandbox snapshot hooks

Do not give built-in tools stronger guarantees than the harness adapter and
sandbox protocol can provide. The same rule applies to analytical
capabilities: if a harness cannot support a required approval, streaming
event, workspace snapshot, structured output mode, or data-access path, the
adapter reports that limit rather than pretend the behavior exists.

## Protocol Before Client

Design protocol behavior before designing the client object. The protocol
needs explicit behavior for start, reconnect, duplicate starts, pending tool
calls, duplicate tool completions, built-in harness events, terminal
results, disconnects, sandbox restarts, workspace upload, skill staging,
snapshot, and rollback.

When these cases stay implicit, clients tend to encode local-only
assumptions. Those assumptions then break the first time the runtime moves
to a real remote sandbox.

## Local Mode

Local mode follows the same split. Application-facing integration still
binds product systems into the definition. Runtime-facing integration still
treats the harness and filesystem as if they could be remote. Local mode is
the remote design in a simpler setting, not a second design.
