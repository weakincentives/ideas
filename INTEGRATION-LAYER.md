# Integration Layer

The integration layer connects an agent definition to a real harness, sandbox,
workspace, tools, skills, and event stream.

A harness adapter is the harness-specific part of that layer. It knows how one
harness wants prompts, tools, skills, approvals, structured output, errors, and
events to look. It does not hide differences between harnesses. It makes them
clear enough to test.

## Why It Exists

A harness is not a neutral shell around a model. The model is trained and
evaluated with the harness's tools, edit loop, files, approvals, transcripts,
and recovery behavior.

The integration layer lets an application use that harness without hard-coding
the whole application to one harness. It keeps the definition stable, gives the
caller named work, routes tool calls, connects the workspace, stages skills,
streams events, checks policies, and records what happened.

## What It Does

The integration layer has a small number of jobs:

- render the definition into harness input
- stage skills and runtime files
- create or reconnect durable work through the sandbox protocol
- attach the right workspace
- route definition tool requests to application handlers
- translate harness events into standard events
- enforce or delegate policies
- validate structured output and completion checks
- distinguish tool failures from transport failures
- emit run records

Business logic belongs in the definition or application. Harness-specific
translation belongs in the harness adapter. Durable work, reconnect, workspace
mounting, and lifecycle rules belong in the sandbox protocol. One codebase may
package these together, but the responsibilities remain separate.

## Harness Adapters

A harness adapter speaks one harness's rules and formats. It knows:

- prompt and message shape
- built-in tool names and event formats
- skill discovery and mount rules
- approval and policy hooks
- structured output support
- terminal result and error formats
- private backend session identifiers

The adapter's job is not to make all harnesses behave the same. Its job is to
preserve the shared definition while making harness differences explicit,
visible, and testable.

## What Belongs Where

Remote sandbox systems usually split responsibility three ways.

**Definition.** Owns prompt structure, declared tools, policies, feedback,
state, resources, completion checks, and output shape.

**Harness adapter.** Translates the definition into one harness's input and
translates harness events back into standard events.

**Sandbox protocol.** Owns durable work, workspace access, lifecycle, reconnect,
isolation, external access policy, and control-plane state.

This split keeps prompts from depending on sandbox lifecycle details and keeps
sandbox code from knowing application intent.

## Bridges

The integration work is broader than one client object.

**Tool bridge.** Routes harness tool requests to application handlers with typed
input, typed output, policy checks, idempotency, and records.

**Skill bridge.** Packages repository, domain, or tool knowledge in the shape
the harness can use.

**Workspace bridge.** Stages files, creates upload tickets, reads and writes
remote files, and exposes snapshots when supported.

**Event bridge.** Converts built-in harness events into standard events while
preserving raw references for debugging.

**Control bridge.** Maps caller-owned work names to sandbox lifecycle,
reconnect, turn ownership, and conflict handling.

**Eval bridge.** Records enough about each run to compare definition, harness
adapter, model, harness, and sandbox changes.

## Protocol Before Client

Design the protocol before designing the client object. The protocol needs
explicit behavior for:

- start versus reconnect
- duplicate start with the same work contract
- duplicate start with a conflicting work contract
- pending tool requests
- duplicate tool completions
- built-in harness events
- terminal results
- harness adapter disconnect
- sandbox restart
- workspace upload
- skill installation or discovery
- snapshot and rollback

When these cases stay implicit, clients tend to encode local-only assumptions.

## Built-In Harness Tools

Every harness has built-in tools. A harness adapter classifies each one:

- unavailable
- available but hidden from the definition
- visible only through observed events
- gated by sandbox policy
- wrapped behind a definition tool
- covered by sandbox snapshot hooks

Do not give built-in tools stronger guarantees than the harness adapter can
provide.

## Contract Tests

The integration layer is ready only when shared tests pass. The tests cover:

- definition rendering
- tool schema translation
- skill packaging
- definition tool call and completion
- policy denial
- feedback delivery
- structured output validation
- built-in event translation
- transaction behavior for supported resources
- workspace upload and read/write
- duplicate turn start
- reconnect with a pending tool call
- disconnect grace expiry
- debug bundle generation

Contract tests run at the protocol layer where possible so implementations in
different languages can share the same expected behavior.

## Local Mode

Local mode follows the same rules:

- caller-owned work IDs
- typed protocol messages, even if in memory
- filesystem abstraction
- streamed events
- explicit built-in tool classification
- skill staging equivalent to remote mode

Local mode is the remote design in a simpler setting, not a second design.
