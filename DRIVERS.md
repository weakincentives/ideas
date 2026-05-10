# Drivers and Integration Points

Drivers bind application-owned definitions and tools to a coupled
model-harness substrate. They are not thin translation layers over a generic
model API.
They are product code at the boundary where application semantics meet harness
semantics.

## Why Drivers Matter

Agentic harnesses are opinionated execution environments. Their models are
trained and evaluated with particular assumptions about tools, files, edit
loops, approval modes, transcripts, and recovery. A useful library should not
hide those assumptions behind a pretend-universal loop.

The driver should expose the harness productively while keeping application
semantics stable:

- caller-owned work identity
- typed tool bridges
- remote workspace binding
- skill packaging
- event normalization
- policy and completion integration
- reconnect and idempotency behavior
- debug and eval artifacts

This is the layer worth investing in.

## Driver Responsibilities

A driver should:

- render the definition
- translate rendered prompt, tool schemas, and skills into harness input
- create or connect durable work
- bind the correct remote workspace
- stream harness events into canonical events
- route definition tool requests to application handlers
- enforce or delegate policies
- propagate budgets and deadlines
- validate structured output
- run completion checks
- distinguish tool errors from transport errors
- emit observability records

The driver should not contain business logic that belongs in the definition or
application. It should contain integration logic that belongs at the
harness/protocol boundary.

## Two Driver Layers

In a remote sandbox architecture there are often two driver layers.

**Definition driver.** Lives with the definition library. It knows how to render
definitions, expose application tools, package skills, and interpret canonical
events.

**Harness driver.** Lives near the sandbox control plane. It knows how to run a
specific harness, translate native events, manage private backend session IDs,
connect the harness to the sandbox protocol, and apply sandbox-local policy.

This split keeps application definitions portable while allowing deep harness
integration.

## Integration Points

The durable surface area is broader than one client class.

- **Tool bridge.** Routes harness tool requests to application handlers with
  typed input, typed output, policy checks, idempotency, and observability.
- **Skill packaging.** Ships repository, domain, or tool-specific operating
  knowledge into the sandbox in the shape the harness can use.
- **Workspace bridge.** Stages files, creates upload tickets, reads and writes
  remote files, and exposes snapshots when supported.
- **Event bridge.** Normalizes native harness events while preserving raw
  references for debugging.
- **Control bridge.** Maps caller-owned work names to sandbox lifecycle,
  reconnect, turn ownership, and conflict handling.
- **Evaluation bridge.** Makes runs reproducible enough to compare driver,
  definition, model, harness, and sandbox changes.

## Protocol Before Client

Design the protocol before designing the client object. The protocol should
make these cases explicit:

- start versus reconnect
- duplicate start with same contract
- duplicate start with conflicting contract
- pending tool request
- duplicate tool completion
- native harness event
- terminal result
- driver disconnect
- sandbox restart
- workspace upload
- skill installation or discovery
- snapshot and rollback

If these cases are left implicit, client libraries tend to encode local-only
assumptions.

## Native Tools

Every harness has native capabilities. Drivers should classify them:

- unavailable
- available but not visible in the definition model
- visible only as observed events
- gated by sandbox policy
- wrapped behind portable tool contracts
- transactional through sandbox snapshot hooks

Do not pretend native tools have stronger semantics than the driver can
actually provide.

## Compatibility Suite

Every driver should pass a shared compatibility suite. The suite should test:

- deterministic render
- tool schema translation
- skill packaging
- definition tool call and completion
- policy denial
- feedback delivery
- structured output validation
- native event normalization
- transaction behavior for supported resources
- workspace upload and read/write
- duplicate turn start
- reconnect with pending tool call
- disconnect grace expiry
- debug bundle generation

Compatibility tests should be protocol-level where possible so multiple
language implementations can share the same behavioral target.

## Local Driver

A local driver is useful. It should still use the same semantics:

- caller-owned work IDs
- typed protocol messages, even if in-memory
- filesystem abstraction
- streamed events
- explicit native-tool capability classification
- skill packaging path equivalent to remote mode

Local mode should be a convenience implementation of the remote contract, not a
different contract.
