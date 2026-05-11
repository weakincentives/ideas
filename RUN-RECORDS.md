# Run Records

Every run needs enough records to reconstruct what happened. A run that cannot
be reconstructed was not observed in any useful sense.

Run records are cross-cutting. They are emitted by all layers, but they should
not become another place to put application intent or harness behavior.

## Records by Source

**Definition records** show what the agent was asked to do:

- rendered definition and prompt hash
- declared tool schemas
- policy and feedback configuration
- structured output shape
- completion checks

**Application-facing records** show how the application participated:

- caller-owned work identifiers
- tool request and completion records
- policy decisions
- resource references
- output delivery records

**Runtime-facing records** show how the definition was run:

- harness adapter name and version
- skill versions and staged file manifests
- built-in harness events
- structured result or terminal error
- raw event references for debugging

**Sandbox and workspace records** show what happened remotely:

- sandbox key and runtime status
- workspace uploads and mutations
- snapshot, commit, and rollback records
- reconnect and ownership events
- budget and deadline records

## Standard Events

Harnesses emit different event streams. Harness adapters convert them into a
standard event shape while preserving raw event references for debugging.

A standard event includes:

- event type
- stable sequence number or timestamp
- tenant or namespace
- sandbox key
- session, thread, and turn identifiers when available
- tool call identifier when applicable
- harness adapter and harness names
- raw event type
- structured event data
- trace context

## Transcripts

The transcript is the human-readable reconstruction of a run. It merges:

- rendered prompt
- model messages
- definition tool calls
- built-in tool events
- feedback
- completion checks
- final output

Transcripts are for inspection. Standard events are the source of truth.

## Debug Bundles

A debug bundle contains enough to reproduce or explain a run:

- definition version and prompt hash
- harness adapter configuration
- skill versions and staged file manifests
- work contract
- event stream
- transcript
- relevant session state snapshots
- workspace manifest or snapshot reference
- policy and completion results
- tool schemas and tool results
- terminal error chain

Bundles avoid embedding secrets. References to remote files or records are fine
when retention and access are clear.

## Trace Propagation

Trace context crosses:

- application layer to runtime-facing integration
- runtime-facing integration to sandbox control plane
- control plane to harness runtime
- harness to provider calls where possible
- definition tool request back to application-facing integration
- tool completion back to harness

Backend provider IDs can be recorded as private correlation fields. Public
correlation uses caller-owned identifiers.

## Retention

Not every record has the same retention requirement. Event summaries may live
longer than raw provider data. Workspace snapshots may expire before
transcripts. Debug bundles record which referenced files or records may expire.

Retention is explicit so evaluation results remain interpretable after large raw
records are cleaned up.
