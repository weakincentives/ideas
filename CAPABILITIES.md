# Capabilities

Capabilities are the declared ways an agent can act. Application and data-system
actions are named tools, not hidden access.

Capabilities sit in the definition. Application-facing integration binds them
to real application handlers. Runtime-facing integration exposes them to the
harness and routes calls back to those handlers.

## Tool Classes

There are two tool classes.

**Definition tools** are declared by the agent definition and fulfilled by
application-facing integration. The definition owns their schema, policy, result
type, and documentation. The application owns the handler and side effects.

**Built-in harness tools** are provided by the harness inside the sandbox.
Common examples include shell, file edit, file read, search, and patch tools.
They are not definition tools unless the harness adapter deliberately wraps them
behind a shared tool shape.

The distinction matters for transactions and records. A definition library can
wrap definition tools. It can only observe built-in harness tools unless the
sandbox protocol exposes control points for policy, snapshot, and rollback.

Skills are separate from tools. Skills explain how to operate; tools perform
application or data-system actions. See [SKILLS.md](SKILLS.md).

## Query and Data Tools

For unattended analytical agents, query execution is a tool surface, not an
untracked sandbox privilege.

A query or data tool should declare:

- query text, query template, or named operation
- typed parameters
- data source, schema, table, or metric references
- authorization and row-level policy checks
- freshness expectations
- row limits, timeouts, and cost limits
- result shape, result handle, or output location
- sampled result rules when full results are too large
- execution ID and trace context

The agent can use skills to learn how to query a data source correctly. The data
access itself still goes through a declared, policy-checked, recorded tool.

## Tool Shape

A tool has:

- stable name
- typed input schema
- typed result schema
- human-readable description
- examples
- visibility or phase rules
- policy hooks
- timeout and budget behavior
- idempotency guidance for external effects

Tool names should fit common protocol limits: lowercase ASCII letters, digits,
underscore, and hyphen are usually safe. Avoid names whose meaning depends on
one harness.

## Tool Results

Tool failure is data, not a system crash. A failed tool returns a typed error
result that the agent can reason about.

Transport failure is different. It means the caller does not know whether the
tool request was received or completed. The caller must reconcile with the
sandbox before retrying.

## Policies, Feedback, and Completion Checks

Policies are fail-closed rules. They can run before render, before a tool call,
after a tool call, before final output, or during reconnect. If a policy cannot
evaluate a request, it denies or stops the action and records why.

Feedback is advisory. It nudges the agent after an event without blocking the
event. Policies prevent invalid behavior; feedback improves behavior that is
still allowed.

Completion checks verify that "done" really means done. They run after the
harness produces a candidate final response and may inspect the session,
workspace metadata, tool history, query history, data freshness, or structured
output.

## External Access

Remote sandboxes restrict application and data-system network access.
Model-provider traffic may be allowed for the harness. Application and data
traffic flows through definition tools fulfilled by application-facing
integration or through explicitly declared sandbox network profiles.

This gives reviewers a clear record:

- every application or data-system action has a tool name
- every tool call has typed input and output
- every query has a data source, execution record, and result reference
- every call can be tied back to stable work identity
- every denial or failure is visible

## Anti-Patterns

- hidden global tool registries
- prompt text that describes unavailable tools
- skills that depend on undeclared files or local paths
- tools whose names or schemas depend on one harness
- policy decisions that are not recorded
- query execution hidden behind broad network access
- broad sandbox network access as a substitute for tool design
- treating built-in harness tools as transactional without protocol support
