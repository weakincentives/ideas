# State and Transactions

State is the durable memory of a run. Transactions say which changes survive
failure. Both need clear limits. State is support machinery around the
definition; it is not the place to encode application business logic or
harness behavior.

The central discipline in this chapter is honesty about rollback. A
transaction should claim to cover only what the definition library actually
controls. Pretending to roll back an external API call, a harness shell
command, or a sandbox mutation that never had a snapshot is worse than
reporting a visible failure: it produces polished outcomes that hide real
side effects.

## Session State

A session is durable state associated with one or more units of work. It
contains typed slices updated by events.

A state slice defines:

- name
- type
- initial value
- reducer
- persistence policy
- rollback policy
- visibility to tools or evaluators

Reducers are pure. They take prior state and an event, then return new
state. They do not call the network, read the clock, or mutate files. That
purity is what makes state rebuildable from the event log.

## Event Log

Events are append-only evidence. They record what happened even when state
rolls back. A failed tool call may leave no committed state change, but the
failure event itself remains.

Useful event families include:

- definition render
- tool declaration
- policy decision
- feedback
- tool request
- tool completion
- built-in harness event
- filesystem snapshot
- query execution
- data freshness check
- review or delivery decision
- transaction result
- completion check
- output

Event design feeds the run records described in
[RUN-RECORDS.md](RUN-RECORDS.md).

## Resources

Resources are injected dependencies with explicit lifetime. Examples include
a filesystem client, database client, object store, clock, sleeper,
credentials provider, or test fixture.

Resources are scoped to a process, session, turn, tool call, or evaluation.
Snapshotable resources may participate in transaction rollback. Other
resources need safe retry behavior or compensating actions.

## Transaction Scope

A transaction can only cover state the definition library controls.

Covered:

- session slices with rollback policy
- snapshotable resources
- remote filesystem mutations when the sandbox protocol supports snapshot
  and restore around the operation
- definition tools routed through the definition library's transaction
  bridge

Not automatically covered:

- external API calls that already completed
- messages already published
- built-in harness file or shell tools unless the sandbox exposes
  transaction hooks
- filesystem writes that bypass the declared filesystem resource
- side effects hidden in finalizers or background tasks

For example, if the harness uses its built-in shell tool to run a script
that writes to a warehouse, the definition library cannot roll that write
back. It can record the event. It can deny the shell tool for that phase
through policy. It cannot undo the mutation after the fact. The right
response is to treat "rollback" as a claim that fails loudly when it is not
supported, not a feature that is silently degraded.

For request-level retry behavior, see [DURABLE-WORK.md](DURABLE-WORK.md).

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
