# Sandboxed Runtimes

Production agent systems usually run the harness in a remote sandbox, not inside
the application process.

This file starts on the runtime side of the agent definition. It describes the
sandbox, runtime, workspace, and protocol that runtime-facing integration talks
to.

## Shape

The usual shape has seven pieces.

```
Application Layer
    |
    | tools, policies, resources
    v
Application-Facing Integration
    |
    | definition inputs
    v
Agent Definition
    |
    | rendered definition
    v
Runtime-Facing Integration / Harness Adapter
    |
    | harness format, skills, events, tool routing
    v
Sandbox Control Plane
    |
    | lifecycle, auth, routing, reconnect, workspace mount, policy
    v
Model-Harness Runtime
    |
    | model loop, built-in tools, provider traffic
    v
Remote Workspace
```

Local development may collapse pieces, but the rules still behave as if the
harness and workspace are remote.

Cloudflare Sandboxes are one concrete example of the runtime side. They provide
an isolated filesystem, dedicated containerized compute, lifecycle operations,
command execution, file operations, and network policy. Those pieces are useful,
but they do not remove the need for an agent-level protocol around work
identity, reconnect, tool routing, workspace persistence, network access, and
run records.

## Application Boundary

The application layer owns the definition inputs and handlers for definition
tools.

The sandbox does not need direct access to application-local paths, processes,
or private services. It reaches application behavior through declared tool
requests.

The same rule applies to analytical systems. Warehouse queries, object-store
reads, dashboard updates, and report publication should enter through declared
tools or explicit network profiles, with records that explain what happened.

## Sandbox Control Plane

The control plane owns sandbox lifecycle and protocol state. It starts and stops
compute, mounts workspaces, authenticates callers, routes messages, tracks
active turns, stores durable ledgers, enforces sandbox policy, and handles
reconnect.

Control-plane state is not definition state. A clear design keeps these records
separate:

- definition and session state owned by the caller-facing library
- durable control-plane records for runtimes, threads, and turns
- live routing state for harness adapters, tool requests, and harnesses
- persistent workspace state in remote storage

## Model-Harness Runtime

The model-harness runtime owns model calls, planning loop, built-in tools, edit
behavior, provider traffic, approvals, and recovery. It may be a CLI harness, a
hosted harness, a protocol-compatible agent runtime, or something not yet
designed.

The runtime can sit behind a stable sandbox protocol without being treated as
identical to other harnesses. Harness adapters absorb the differences and make
them explicit.

The definition library does not assume the harness can call in-process
functions or read local files from the application layer.

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

The protocol uses stable public identifiers for retryable work. Backend runtime
identifiers can exist, but they stay private to harness adapters and the control
plane.

## External Access

Remote sandboxes default to restricted network access.

- Provider and model traffic can be allowed through a narrow profile.
- Application external access uses definition tools.
- Setup-time network access is explicit and auditable.
- Broad network access is an exception, not the default runtime mode.

This makes the protocol tool-oriented rather than network-oriented.

## Local Mode

Local mode supports tests and development. It follows the same rules:

- stable idempotency keys
- work contract conflicts
- filesystem abstraction
- streamed events
- tool request and completion messages
- reconnect behavior where practical

Avoid writing a local-path implementation that later needs a different remote
design.
