# Integration Layer

The integration layer binds application-owned definitions, tools, skills, and
work identifiers to model-harness runtimes and sandbox protocols. It is where
portable application behavior meets harness-specific behavior.

`Harness adapter` names the harness-specific part of that layer. It is not the
architecture. It is the component that translates the shared integration
contract into one harness's prompt shape, built-in tools, event stream, skill
format, approval behavior, structured output support, and error behavior.

## Why the Layer Matters

Agentic harnesses are opinionated execution environments. Their models are
trained and evaluated with assumptions about tools, files, edit loops, approval
modes, transcripts, and recovery. A definition library preserves those
assumptions instead of flattening the harness into a generic loop.

The integration layer keeps application behavior stable while exposing the
harness in the way it works best:

- caller-owned work identity
- typed tool bridges
- remote workspace binding
- skill packaging
- harness adapter behavior
- event normalization
- policy and completion integration
- reconnect and idempotency behavior
- debug and eval records

## Integration Responsibilities

The integration layer:

- renders the definition
- translates rendered prompt, tool schemas, and skills into harness input
- creates or connects durable work through the sandbox protocol
- binds the correct remote workspace
- streams harness events into standard events
- routes definition tool requests to application handlers
- enforces or delegates policies
- propagates budgets and deadlines
- validates structured output
- runs completion checks
- distinguishes tool errors from transport errors
- emits evidence records

Business logic belongs in the definition or application. Harness-specific
translation belongs in the harness adapter. Durable work, reconnect, workspace
mounting, and lifecycle rules belong in the sandbox protocol, even when one
implementation packages those pieces together.

## Harness Adapters

A harness adapter speaks one harness's rules and formats. It knows:

- prompt and message shape
- built-in tool vocabulary
- edit and filesystem event formats
- skill discovery and mounting conventions
- approval and policy integration points
- structured output support
- harness error and terminal result formats
- private backend session identifiers

The harness adapter's job is not to make all harnesses behaviorally identical.
Its job is to make differences explicit, observable, and testable while
preserving the portable definition contract.

## Integration Boundaries

Remote sandbox architectures usually have three boundaries.

**Definition boundary.** Owns portable intent: prompt structure, tools,
policies, state, resources, completion checks, and output contracts.

**Harness adapter boundary.** Translates the integration contract into a
specific harness and translates harness events back into standard events.

**Sandbox protocol boundary.** Owns durable work, workspace access, lifecycle,
reconnect, isolation, external access policy, and control-plane state.

This split keeps definitions portable, lets harness adapters go deep on harness
behavior, and prevents sandbox lifecycle from leaking into prompt design.

## Integration Points

The integration work is broader than one client class.

- **Tool bridge.** Routes harness tool requests to application handlers with
  typed input, typed output, policy checks, idempotency, and records.
- **Skill packaging.** Ships repository, domain, or tool-specific operating
  knowledge into the sandbox in the shape the harness can use.
- **Workspace bridge.** Stages files, creates upload tickets, reads and writes
  remote files, and exposes snapshots when supported.
- **Event bridge.** Normalizes built-in harness events while preserving raw
  references for debugging.
- **Control bridge.** Maps caller-owned work names to sandbox lifecycle,
  reconnect, turn ownership, and conflict handling.
- **Evaluation bridge.** Makes runs reproducible enough to compare definition,
  harness adapter, model, harness, and sandbox changes.

## Protocol Before Client

Design the protocol before designing the client object. The protocol makes
these cases explicit:

- start versus reconnect
- duplicate start with same contract
- duplicate start with conflicting contract
- pending tool request
- duplicate tool completion
- built-in harness event
- terminal result
- harness adapter disconnect
- sandbox restart
- workspace upload
- skill installation or discovery
- snapshot and rollback

If these cases are left implicit, client libraries tend to encode local-only
assumptions.

## Built-In Tools

Every harness has built-in capabilities. Harness adapters classify them:

- unavailable
- available but not visible in the definition model
- visible only as observed events
- gated by sandbox policy
- wrapped behind portable tool contracts
- transactional through sandbox snapshot hooks

Do not give built-in tools stronger guarantees than the harness adapter can
actually provide.

## Contract Tests

Every integration layer passes a shared contract test suite. The suite tests:

- deterministic render
- tool schema translation
- skill packaging
- definition tool call and completion
- policy denial
- feedback delivery
- structured output validation
- built-in event normalization
- transaction behavior for supported resources
- workspace upload and read/write
- duplicate turn start
- reconnect with pending tool call
- disconnect grace expiry
- debug bundle generation

Contract tests run at the protocol layer where possible so multiple language
implementations share the same expected behavior.

## Local Integration

Local integration uses the same rules:

- caller-owned work IDs
- typed protocol messages, even if in-memory
- filesystem abstraction
- streamed events
- explicit built-in tool capability classification
- skill packaging path equivalent to remote mode

Local mode is a convenience implementation of the remote contract, not a second
architecture.
