# Sandboxed Runtimes

Production analytical agents usually run the harness in a sandbox, not inside
the application process. This chapter starts on the runtime side of the
agent definition and describes the sandbox, runtime, workspace, and protocol
that runtime-facing integration talks to.

The point of running the harness remotely is not isolation for its own sake.
It is that the harness can take long-running, expensive, privileged actions
on behalf of the application without the application having to grant the
harness broad local access. Remote-first design forces the real boundaries
into view: work identity, tool routing, workspace persistence, data access,
and egress control.

## Shape

The usual shape has these pieces:

```
Application Layer
    |
    | tools, policies, resources, output sinks
    v
Application-Facing Integration
    |
    | definition inputs
    v
Agent Definition
    |
    | rendered definition and requirements
    v
Runtime-Facing Integration / Harness Adapter
    |
    | harness format, skills, events, tool routing
    v
Sandbox Control Plane
    |
    | lifecycle, auth, reconnect, workspace mount, policy
    v
Model-Harness Runtime
    |
    | model loop, built-in tools, provider traffic
    v
Remote Workspace
```

Local development may collapse some of these pieces, but the rules should
still behave as if the harness and workspace are remote.

Cloudflare Sandboxes are one concrete example. They can provide filesystem
isolation, dedicated compute, lifecycle operations, command execution, file
operations, and granular network policy. Those pieces are useful building
blocks, but they do not replace the agent-level protocol for work identity,
reconnect, tool routing, workspace persistence, data access, egress control,
and run records. The sandbox gives you the runtime; the agent system still
has to give you the contract the runtime runs under.

## Application Boundary

The application owns the definition inputs and the handlers for definition
tools. The sandbox should not need direct access to application-local paths,
processes, or private services. It reaches application behavior through
declared tool requests.

The same rule applies to analytical systems. Warehouse queries, object-store
reads, dashboard updates, and report publication should enter through
declared tools or explicit network profiles, with records that explain what
happened.

## Sandbox Control Plane

The control plane owns sandbox lifecycle and protocol state. It starts and
stops compute, mounts workspaces, authenticates callers, routes messages,
tracks active turns, stores runtime ledgers, enforces sandbox policy, and
handles reconnect.

Control-plane state is not definition state. A clean design keeps these
records separate:

- definition and session state owned by the caller-facing library
- durable control-plane records for runtimes, threads, and turns
- live routing state for harness adapters, tool requests, and harnesses
- persistent workspace state in remote storage

Mixing them produces designs where a sandbox restart loses application
intent, or where a library upgrade is forced to migrate runtime routing
state.

## Model-Harness Runtime

The model-harness runtime owns model calls, the model loop, built-in tools,
edit behavior, provider traffic, approvals, and recovery. It may be a CLI
harness, a hosted harness, a protocol-compatible runtime, or something not
yet designed.

The runtime can sit behind a stable sandbox protocol without being treated
as identical to other harnesses. Harness adapters absorb the differences and
make them explicit. The definition library should not assume the harness
can call in-process functions or read local files from the application
layer.

## Protocol Shape

A sandbox protocol needs operations for:

- start sandbox
- stop sandbox
- recover sandbox
- inspect runtime status
- create or connect thread
- start or resume turn
- stream events
- request definition tool execution
- complete definition tool execution
- read and write workspace files
- create upload tickets for large files
- snapshot and restore workspace state
- list built-in skills or capabilities
- stage or mount skills provided by harness adapters
- close, detach, and reattach connections

The protocol uses stable public identifiers for retryable work. Backend
runtime identifiers can exist, but they stay private to harness adapters and
the control plane. See [DURABLE-WORK.md](DURABLE-WORK.md) for the identity
and retry rules these operations enforce.

## External Access

Remote sandboxes default to restricted network access:

- Provider and model traffic can be allowed through a narrow profile.
- Application external access uses definition tools.
- Data-system access uses declared tools or explicit network profiles.
- Setup-time network access is explicit and recorded.
- Broad network access is an exception, not the default runtime mode.

This makes the protocol tool-oriented rather than network-oriented, which is
what [CAPABILITIES.md](CAPABILITIES.md) relies on to keep every
application-visible action named and traced.

## Local Mode

Local mode supports tests and development. It follows the same rules:

- stable work identifiers
- work contract conflicts
- filesystem abstraction
- streamed events
- tool request and completion messages
- reconnect behavior where practical

Avoid writing a local-path implementation that later needs a different
remote design. The workspace rules that apply here live in
[WORKSPACES.md](WORKSPACES.md).
