# State and Transactions

State is the durable memory of a run. Transactions are scoped guarantees about
which mutations survive failure. Both need clear boundaries.

## Session State

A session is caller-named state associated with one or more units of work. It
contains typed slices updated by events.

A state slice should define:

- name
- type
- initial value
- reducer
- persistence policy
- rollback policy
- visibility to tools or evaluators

Reducers should be pure. They take prior state and an event, then return new
state. They do not call the network, read the clock, or mutate files.

## Event Log

Events are append-only evidence. They record what happened even when state rolls
back. A failed tool call may leave no committed state mutation, but the failure
event itself should remain.

Useful event families include:

- definition rendered
- tools declared
- policy evaluated
- feedback emitted
- tool requested
- tool completed
- native harness event observed
- filesystem snapshot created
- transaction committed or rolled back
- completion checked
- output produced

## Resources

Resources are injected dependencies with explicit lifetime. Examples include a
filesystem client, database client, object store, clock, sleeper, credentials
provider, or test fixture.

Resources should be scoped:

- process singleton
- session
- turn
- tool call
- evaluation

Snapshotable resources may participate in transaction rollback. Non-snapshotable
resources need idempotency keys or compensating actions.

## Transaction Scope

A transaction can only cover state the framework controls.

Covered:

- session slices with rollback policy
- snapshotable resources
- remote filesystem mutations when the sandbox protocol supports snapshot and
  restore around the operation
- definition tools routed through the framework's transaction bridge

Not automatically covered:

- external API calls that already completed
- messages already published
- native harness file or shell tools unless the sandbox exposes transaction
  hooks
- filesystem writes that bypass the declared filesystem resource
- side effects hidden in finalizers or background tasks

## Tool Transactions

For a definition tool, the normal flow is:

1. validate input
2. evaluate pre-call policies
3. snapshot session and snapshotable resources
4. execute handler
5. validate result
6. commit on success
7. rollback and return typed failure on error
8. record events either way

The agent should see a tool result, not transaction machinery.

## Request Idempotency

Transactionality inside a turn is not enough. The turn itself needs
idempotency. A durable request should be identified by:

- caller-owned work identifier
- caller-owned turn identifier
- semantic contract hash

The semantic contract should include inputs and declarations that change model
behavior: prompt hash, tool schemas, model settings, structured output schema,
workspace reference, and relevant policy configuration.

Operational envelope values such as wait timeout or client connection deadline
should only be part of the semantic contract when they alter sandbox behavior.
Otherwise they are retry parameters, not a reason to create a conflict.

## Native Tool Limits

Many harnesses have native file, command, search, and patch tools. A definition
library should not overclaim control over them. There are three honest levels:

- observe native tool events and include them in transcripts
- gate native tools through sandbox policy
- wrap native tools in transaction hooks exposed by the sandbox protocol

Only the third level supports full rollback semantics.
