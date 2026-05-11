# Run Records

Every run needs enough records to reconstruct what happened. A run that
cannot be reconstructed was not observed in any useful sense. That failure
is most visible in analytical work, where a polished answer may be
challenged weeks later: if the query IDs, intermediate files, freshness
checks, and policy decisions are gone, there is no way to defend or correct
the result.

Run records are emitted by every layer in the system. They should not
become another place to put application intent or harness behavior. They
are evidence, not configuration.

## Records by Layer

**Definition records** show what the agent was asked to do:

- rendered definition and prompt hash
- declared tool schemas
- required skills
- policy and feedback configuration
- structured output shape
- completion checks

**Application-facing records** show how the application participated:

- stable work identifiers
- tool request and completion records
- query execution records
- data source, schema, metric, or dataset references
- data freshness and permission decisions
- cost, review, escalation, and delivery decisions
- policy decisions
- resource references
- output delivery records

**Runtime-facing records** show how the definition was run:

- harness adapter name and version
- skill versions and staged file manifests
- built-in harness events
- generated file and report references
- structured result or terminal error
- raw event references for debugging

**Sandbox and workspace records** show what happened remotely:

- sandbox key and runtime status
- workspace uploads and mutations
- snapshot, commit, and rollback records
- reconnect and ownership events
- budget and deadline records

The state that feeds these records is defined in [STATE.md](STATE.md);
records and state are separate surfaces but read from the same event log.

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
- query summaries or result references
- built-in tool events
- feedback
- completion checks
- final output

Transcripts are for inspection. Standard events are the source of truth.

## Debug Bundles

A debug bundle contains enough to reproduce or explain a run. For example,
when a weekly cohort report is challenged three weeks later, a good debug
bundle lets a reviewer answer the obvious questions without waking anyone
up: which metric version was used, whether freshness passed, which query
IDs produced which numbers, whether review approved delivery, and which
skill version was staged.

A bundle contains:

- definition version and prompt hash
- harness adapter configuration
- skill versions and staged file manifests
- work contract
- event stream
- transcript
- relevant session state snapshots
- workspace manifest or snapshot reference
- query text or redacted query reference
- query execution IDs and result references
- data source, schema, metric, or dataset versions
- freshness checks and access decisions
- cost, review, and delivery decisions
- policy and completion results
- validation checks
- tool schemas and tool results
- generated outputs
- terminal error chain

Bundles avoid embedding secrets. References to remote files or records are
fine when retention and access are clear.

## Trace Links

Trace context crosses:

- application layer to runtime-facing integration
- runtime-facing integration to sandbox control plane
- control plane to harness runtime
- harness to provider calls where possible
- definition tool request back to application-facing integration
- tool completion back to harness

Backend provider IDs can be recorded as private correlation fields. Public
correlation uses stable work identifiers, as described in
[DURABLE-WORK.md](DURABLE-WORK.md).

## Retention

Not every record has the same retention requirement. Event summaries may
live longer than raw provider data. Workspace snapshots may expire before
transcripts. Debug bundles record which referenced files or records may
expire.

Retention is explicit so evaluation results remain interpretable after
large raw records are cleaned up. See [EVALUATION.md](EVALUATION.md) for
how records feed evaluators.

For analytical work, useful evidence is often sensitive. Records may need
redaction, sampling, encryption, access control, short raw-data retention,
references instead of embedded data, and deletion rules. A run should
remain reviewable after sensitive raw data expires, even if it cannot be
fully replayed.
