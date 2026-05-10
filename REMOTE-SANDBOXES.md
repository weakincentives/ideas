# Sandboxed Runtimes

Production agent systems assume the harness runs in a remote sandbox, not
inside the application process.

## Topology

The usual topology has four pieces.

```
Application / Orchestrator
    |
    | definition, driver, skills, durable work IDs, tool handlers
    v
Sandbox Control Plane
    |
    | lifecycle, auth, routing, reconnect, workspace mount, policy
    v
Execution Substrate
    |
    | model loop, native tools, provider traffic
    v
Remote Workspace
```

The implementation may collapse pieces for local development, but the contract
still behaves as if the boundary is remote.

## Orchestrator

The orchestrator owns the definition and caller-visible work identity. It
renders prompts, declares tools, fulfills definition tool calls, evaluates
policies that live in the definition layer, records application-level events,
and decides when to start, retry, reattach, or abandon work.

The orchestrator does not require direct access to sandbox-local paths or
processes.

## Sandbox Control Plane

The control plane owns sandbox lifecycle and protocol state. It starts and
stops compute, mounts workspaces, authenticates callers, routes messages,
tracks active turns, stores durable ledgers, enforces sandbox policy, and
mediates reconnect.

Control-plane state is not the same as definition state. A good design keeps at
least four tiers separate:

- definition and session state owned by the caller-facing framework
- durable control-plane records such as runtime, thread, and turn ledgers
- live routing state needed to connect drivers, tool requests, and harnesses
- persistent workspace state in remote storage

## Execution Substrate

The execution substrate owns the coupled model-harness behavior: model calls,
planning loop, native tools, edit semantics, provider traffic, approval modes,
and runtime recovery. It may be a CLI harness, service-hosted harness,
protocol-compatible agent runtime, or something not yet designed.

The substrate can sit behind a stable sandbox protocol without being treated as
behaviorally identical to other harnesses. Drivers absorb those differences and
make them explicit.

The definition library does not assume the harness can call in-process functions
or read local files from the orchestrator.

## Protocol Surface

A portable sandbox protocol needs primitives for:

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
- list native skills or capabilities
- stage or mount driver-provided skills
- close, detach, and reattach connections

The protocol uses caller-owned identifiers in public payloads. Backend runtime
identifiers can exist, but they stay private to drivers and the control plane.

## Egress

Remote sandboxes default to restricted egress.

- Provider and model traffic can be allowed through a narrow profile.
- Application egress uses definition tools.
- Setup-time egress is explicit and auditable.
- Broad egress is an exception, not the default runtime mode.

This makes the protocol capability-oriented rather than network-oriented.

## Local Mode

Local mode supports tests and development. It emulates the same protocol
semantics:

- caller-owned work IDs
- semantic contract conflicts
- filesystem abstraction
- streamed events
- tool request and completion messages
- reconnect behavior where practical

Avoid writing a local path based implementation that later needs a different
remote architecture.
