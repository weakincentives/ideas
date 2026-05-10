# Durable Work

Durable work is the set of semantics that make long-running agent operations
safe under retry, reconnect, and sandbox restart.

## Three Lifecycles

Keep these lifecycles separate.

**Compute lifecycle.** A sandbox or harness process starts, sleeps, restarts,
or stops.

**Work lifecycle.** A caller-named session, thread, turn, or evaluation exists,
accumulates state, reaches a terminal result, or is deleted.

**Connection lifecycle.** A client connects, streams messages, disconnects, and
possibly reconnects.

A disconnect is not cancellation. A restart is not deletion. A reconnect is not
new work.

## Public Identity

The caller should name durable work. Typical identifiers:

- tenant or namespace
- sandbox key
- session key
- thread key
- turn key
- workspace key

Names do not need to be globally meaningful. They need to be stable inside the
tenant scope and supplied by the caller that wants idempotency.

Backend identifiers may exist internally. They should not be required public
handles.

## Semantic Contract

A turn or evaluation should carry a semantic contract hash. Include values that
change behavior:

- user input
- rendered prompt hash
- declared tool schemas
- model and effort settings
- structured output schema
- workspace reference
- relevant policies
- relevant memory or transcript replay settings

Do not include values that only change client waiting behavior unless they
change sandbox execution. For example, a shorter client wait timeout should not
usually make the same turn conflict with itself.

## Idempotent Start

Starting work with the same public identifier and same semantic contract should
return the current or completed work. It must not double-execute.

Starting work with the same identifier and a different semantic contract should
return an explicit conflict.

Starting work with a different identifier is new work, even if the content is
identical.

## Reattach

After a disconnect, the sandbox control plane should keep useful work alive for
a bounded grace window. Reattach should return enough state for the caller to
resume:

- runtime status
- active thread or turn
- pending definition tool calls
- recent terminal turn result
- last observed sequence number
- workspace reference
- conflict or ownership status

The exact timeout is an operational choice. The semantic requirement is that
transient transport failure does not force duplicate work.

## Tool Requests

When the harness requests a definition tool, the sandbox protocol should route
the request to the connected caller or an equivalent tool executor.

Tool requests need stable call identifiers. Completion should be idempotent:
resubmitting the same tool result after reconnect should be accepted or safely
deduplicated.

If the caller never reconnects, pending tool calls should fail with a typed
reason, and the turn should reach a forensics-friendly terminal state.

## Ownership

Only one live connection should normally own a mutable turn. Multiple readers
may observe, but one driver should fulfill pending tool calls. Ownership changes
should be explicit and recorded.

## Deletion

Deleting work is separate from stopping compute. A caller may stop the sandbox
without deleting the workspace or turn ledger. A caller may delete a session
without terminating the whole sandbox. APIs should make the scope clear.
