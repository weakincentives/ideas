# Integration Layer

Drivers bind application-owned definitions, tools, skills, and work identifiers
to a coupled execution substrate. They are product code at the boundary where
application semantics meet harness semantics.

## Why Drivers Matter

Agentic harnesses are opinionated execution environments. Their models are
trained and evaluated with assumptions about tools, files, edit loops, approval
modes, transcripts, and recovery. A definition library preserves those
assumptions instead of flattening the harness into a generic loop.

The driver exposes the harness while keeping application semantics stable:

- caller-owned work identity
- typed tool bridges
- remote workspace binding
- skill packaging
- event normalization
- policy and completion integration
- reconnect and idempotency behavior
- debug and eval artifacts

## Driver Responsibilities

A driver:

- renders the definition
- translates rendered prompt, tool schemas, and skills into harness input
- creates or connects durable work
- binds the correct remote workspace
- streams harness events into canonical events
- routes definition tool requests to application handlers
- enforces or delegates policies
- propagates budgets and deadlines
- validates structured output
- runs completion checks
- distinguishes tool errors from transport errors
- emits observability records

The driver contains integration logic at the harness/protocol boundary, not
business logic that belongs in the definition or application.

## Two Integration Boundaries

In a remote sandbox architecture there are often two integration boundaries.

**Definition-side driver.** Lives with the definition library. It knows how to
render definitions, expose application tools, package skills, and interpret
canonical events.

**Harness-side driver.** Lives near the sandbox control plane. It knows how to
run a specific harness, translate native events, manage private backend session
IDs, connect the harness to the sandbox protocol, and apply sandbox-local
policy.

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

Design the protocol before designing the client object. The protocol makes
these cases explicit:

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

Every harness has native capabilities. Drivers classify them:

- unavailable
- available but not visible in the definition model
- visible only as observed events
- gated by sandbox policy
- wrapped behind portable tool contracts
- transactional through sandbox snapshot hooks

Do not give native tools stronger semantics than the driver can actually
provide.

## Conformance Suite

Every driver passes a shared conformance suite. The suite tests:

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

Conformance tests are protocol-level where possible so multiple language
implementations share the same behavioral target.

## Local Driver

A local driver uses the same semantics:

- caller-owned work IDs
- typed protocol messages, even if in-memory
- filesystem abstraction
- streamed events
- explicit native-tool capability classification
- skill packaging path equivalent to remote mode

Local mode is a convenience implementation of the remote contract, not a
different contract.
