# Durable Work

Durable work survives retry, reconnect, and sandbox restart. For unattended
analytical agents, this matters because work may be expensive, long-running,
and hard to repeat safely. A disconnected client should not force a query,
code run, or report build to start over by accident.

The shape of the problem is less about distributed systems theory and more
about three lifecycles that look alike and behave differently. Conflating
them is the most common source of lost work and duplicate work.

## Three Lifecycles

Keep these lifecycles separate.

**Compute lifecycle.** A sandbox or harness process starts, sleeps,
restarts, or stops.

**Work lifecycle.** A session, thread, turn, or evaluation exists,
accumulates state, reaches a terminal result, or is deleted.

**Connection lifecycle.** A client connects, streams messages, disconnects,
and possibly reconnects.

A disconnect is not cancellation. A restart is not deletion. A reconnect is
not new work.

## Public Identity

Durable work uses stable public identifiers supplied by the caller. Typical
identifiers:

- tenant or namespace
- sandbox key
- session key
- thread key
- turn key
- workspace key

Names do not need to be globally meaningful. They need to be stable inside
the tenant scope and supplied by the caller that wants safe retries. Backend
identifiers may exist internally; they are not public handles. For the
runtime operations that enforce these identifiers, see
[REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md).

## Work Contract

A turn or evaluation carries a work contract hash. Include values that
change behavior:

- user input
- rendered prompt hash
- declared tool schemas
- model and effort settings
- structured output schema
- workspace reference
- relevant policies
- relevant skills
- data sources or metric versions that affect results
- relevant memory or transcript replay settings

Do not include values that only change client waiting behavior unless they
change sandbox execution. For example, a shorter client wait timeout does
not usually make the same turn conflict with itself.

## Safe Start

Starting work with the same public identifier and same work contract
returns the current or completed work. It must not double-execute. Starting
work with the same identifier and a different work contract returns an
explicit conflict. Starting work with a different identifier is new work,
even if the content is identical.

## Reattach

After a disconnect, the sandbox control plane keeps work alive for a bounded
grace window. Reattach returns enough state for the caller to resume:

- runtime status
- active thread or turn
- pending definition tool calls
- recent terminal turn result
- last observed sequence number
- workspace reference
- conflict or ownership status

For example: a caller submits a warehouse query through a definition tool,
then its network drops. The sandbox keeps the turn alive, the query
continues, and the caller reconnects two minutes later. Reattach returns
the active turn, the pending tool call for the query, and the last event
sequence number. The caller resumes streaming; the query does not run
twice. The exact grace timeout is an operational choice, but the property
is non-negotiable: transient transport failure does not force duplicate
work.

## Tool Requests

When the harness requests a definition tool, the sandbox protocol routes
the request to the connected caller or an equivalent tool executor. Tool
requests need stable call identifiers. Completion is retry-safe:
resubmitting the same tool result after reconnect is accepted or safely
ignored.

If the caller never reconnects, pending tool calls fail with a typed
reason, and the turn reaches a terminal state that can be inspected later.

## Ownership

Only one live connection normally owns a mutable turn. Multiple readers may
observe, but one harness adapter fulfills pending tool calls. Ownership
changes are explicit and recorded.

## Deletion

Deleting work is separate from stopping compute. A caller may stop the
sandbox without deleting the workspace or turn ledger. A caller may delete a
session without terminating the whole sandbox. APIs make the scope clear.
For the state and transaction implications of deletion, see
[STATE.md](STATE.md).
