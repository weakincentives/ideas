# Observability

Observability is part of the contract. A run that cannot be reconstructed was
not really observed.

## Required Artifacts

A useful system should emit:

- rendered definition and prompt hash
- declared tool schemas
- policy decisions
- feedback records
- tool request and completion records
- native harness event records
- workspace upload and mutation records
- transaction snapshot, commit, and rollback records
- completion check records
- final output or terminal error
- budget and deadline records
- trace identifiers across protocol boundaries

These records should use caller-owned work identifiers.

## Canonical Events

Harnesses emit different event streams. Drivers should normalize them into a
canonical event envelope while preserving raw payload references for debugging.

A canonical event should include:

- event type
- stable sequence number or timestamp
- tenant or namespace
- sandbox key
- session, thread, and turn identifiers when available
- tool call identifier when applicable
- driver and harness names
- raw event type
- structured payload
- trace context

## Transcripts

The transcript is the human-readable reconstruction of a run. It should merge:

- rendered prompt
- model messages
- definition tool calls
- native tool events
- feedback
- completion checks
- final output

Transcripts are for inspection. Canonical events are the source of truth.

## Debug Bundles

A debug bundle should contain enough to reproduce or explain a run:

- definition version and prompt hash
- driver configuration
- skill versions and staged file manifests
- semantic contract
- event stream
- transcript
- relevant session state snapshots
- workspace manifest or snapshot reference
- policy and completion results
- tool schemas and tool results
- terminal error chain

Bundles should avoid embedding secrets. References to remote artifacts are
acceptable when retention and access are clear.

## Trace Propagation

Trace context should cross:

- orchestrator to sandbox control plane
- control plane to harness runtime
- harness to provider calls where possible
- sandbox egress proxy
- definition tool request back to orchestrator
- tool completion back to harness

Backend provider IDs can be recorded as private correlation fields. Public
correlation should use caller-owned identifiers.

## Retention

Not every artifact has the same retention requirement. Event summaries may live
longer than raw provider payloads. Workspace snapshots may expire before
transcripts. Debug bundles should record which referenced artifacts may expire.

Retention should be explicit so evaluation results remain interpretable after
large raw artifacts are cleaned up.
