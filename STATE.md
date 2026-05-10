# State, Transactions, and Idempotency

State is the durable memory of a run. Transactions say which changes survive
failure. Both need clear limits.

## Session State

A session is caller-named state associated with one or more units of work. It
contains typed slices updated by events.

A state slice defines:

- name
- type
- initial value
- reducer
- persistence policy
- rollback policy
- visibility to tools or evaluators

Reducers are pure. They take prior state and an event, then return new state.
They do not call the network, read the clock, or mutate files.

## Event Log

Events are append-only evidence. They record what happened even when state rolls
back. A failed tool call may leave no committed state change, but the failure
event itself remains.

Useful event families include:

- definition rendered
- tools declared
- policy evaluated
- feedback emitted
- tool requested
- tool completed
- built-in harness event observed
- filesystem snapshot created
- transaction committed or rolled back
- completion checked
- output produced

## Resources

Resources are injected dependencies with explicit lifetime. Examples include a
filesystem client, database client, object store, clock, sleeper, credentials
provider, or test fixture.

Resources are scoped:

- process singleton
- session
- turn
- tool call
- evaluation

Snapshotable resources may participate in transaction rollback. Other resources
need idempotency keys or compensating actions.

## Transaction Scope

A transaction can only cover state the definition library controls.

Covered:

- session slices with rollback policy
- snapshotable resources
- remote filesystem mutations when the sandbox protocol supports snapshot and
  restore around the operation
- definition tools routed through the definition library's transaction bridge

Not automatically covered:

- external API calls that already completed
- messages already published
- built-in harness file or shell tools unless the sandbox exposes transaction
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

The agent sees a tool result, not transaction machinery.

## Request Idempotency

Transactionality inside a turn is not enough. The turn itself needs idempotency.
A durable request is identified by:

- caller-owned work identifier
- caller-owned turn identifier
- work contract hash

The work contract includes inputs and declarations that change model behavior:
prompt hash, tool schemas, model settings, structured output schema, workspace
reference, and relevant policy configuration.

Operational values such as wait timeout or client connection deadline belong in
the work contract only when they alter sandbox behavior. Otherwise they are
retry parameters, not a reason to create a conflict.

## Built-In Tool Limits

Many harnesses have built-in file, command, search, and patch tools. A
definition library must not overclaim control over them. There are three honest
levels:

- observe built-in tool events and include them in transcripts
- gate built-in tools through sandbox policy
- wrap built-in tools in transaction hooks exposed by the sandbox protocol

Only the third level supports full rollback guarantees.
