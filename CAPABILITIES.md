# Capabilities

Capabilities are the declared ways an agent can act. They sit in the
definition, then get connected to the application and runtime by integration
code. The central rule is simple: application and data-system actions are
named tools, not hidden sandbox access. A declared tool is something reviewers
can see, budget, rate-limit, and trace. Broad network access inside the
sandbox is none of those things.

## Two Tool Classes

**Definition tools** are declared by the agent definition and fulfilled by
application-facing integration. The definition owns their schema, policy,
result type, and documentation. The application owns the handler and side
effects.

**Built-in harness tools** are provided by the harness inside the sandbox.
Common examples include shell, file edit, file read, search, patch, and
native query helpers. They are not definition tools unless the harness
adapter deliberately wraps them behind a shared tool shape.

The distinction matters for transactions and records. A definition library
can wrap definition tools. It can only observe built-in harness tools unless
the sandbox protocol exposes control points for policy, snapshot, and
rollback. Skills are a separate surface: skills explain how to operate, while
tools perform application or data-system actions (see
[SKILLS.md](SKILLS.md)).

## Analytical Job Tools

For unattended analytical agents, query execution is a tool surface. It
should not be an untracked privilege inside the sandbox.

A query, data, or analytical job tool should declare:

- query text, query template, or named operation
- typed parameters
- data source, schema, table, or metric references
- authorization and row-level checks
- freshness expectations
- row limits, timeouts, and cost limits
- result shape, result handle, or output location
- sampled result rules when full results are too large
- async job status and cancellation behavior
- partial result behavior
- materialized intermediate outputs
- notebook, report, dashboard, or output references
- warehouse or session settings that affect results
- execution ID and trace context

For example, a `run_warehouse_query` tool might accept a query template name
and typed parameters, reference a metric version with a stated freshness
rule, enforce a row-limit and a cost budget, and return a typed result handle
rather than inline rows. The agent sees a handle; the run record captures the
query ID, the metric version, the freshness check, and the result size.
Later, a reviewer can trace the final answer back through the handle to the
exact execution.

Analytical tools are rarely simple request-response calls. A query may launch
a job, produce intermediate tables, stream partial results, write files,
update a dashboard, or require later inspection. The tool shape should expose
that lifecycle rather than hide it behind one opaque result.

## Data Meaning

Query correctness depends on more than valid SQL. The agent needs to know
what the data means. Definition tools, skills, and records should make these
visible:

- metric definitions and owners
- schema versions and table ownership
- freshness and late-arriving-data rules
- row-level filters and privacy filters
- slowly changing dimensions
- sampling rules
- warehouse-specific settings that affect results

The agent should not infer business meaning from table names alone. A column
called `revenue` may exclude refunds in one schema and include them in
another. A dashboard that fails to name the rule will produce a confident
wrong answer.

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
- retry guidance for external effects

Tool names should fit common protocol limits: lowercase ASCII letters,
digits, underscore, and hyphen are usually safe. Avoid names whose meaning
depends on one harness.

## Tool Results

Tool failure is data, not a system crash. A failed tool returns a typed error
result that the agent can reason about. Transport failure is different: it
means the caller does not know whether the tool request was received or
completed. The caller must reconcile with the sandbox before retrying. For
the safe-retry rules that govern this reconciliation, see
[DURABLE-WORK.md](DURABLE-WORK.md).

## Policies, Feedback, and Completion Checks

Policies are fail-closed rules. They can run before render, before a tool
call, after a tool call, before final output, or during reconnect. If a
policy cannot evaluate a request, it denies or stops the action and records
why.

Feedback is advisory. It nudges the agent after an event without blocking
the event. Policies prevent invalid behavior; feedback improves behavior that
is still allowed.

Completion checks verify that "done" really means done. They run after the
harness produces a candidate final response and may inspect the session,
workspace metadata, tool history, query history, data freshness, generated
outputs, or structured output.

## External Access

Remote sandboxes restrict application and data-system network access. Model
provider traffic may be allowed through a narrow profile for the harness.
Application and data traffic flows through definition tools fulfilled by
application-facing integration, or through explicitly declared sandbox
network profiles. For the runtime boundary that enforces this, see
[REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md).

The payoff for reviewers is concrete:

- every application or data-system action has a tool name
- every tool call has typed input and output
- every query has a data source, execution record, and result reference
- every denial or failure is visible
- every final output can be tied back to work identity and evidence

## Rollback Honesty

A definition tool routed through the library's transaction bridge can be
rolled back. A built-in harness edit or shell command cannot, unless the
sandbox protocol supports snapshot and restore around it. Do not claim
stronger guarantees than the runtime can deliver. See [STATE.md](STATE.md)
for the transaction scope that applies to tools.

## Anti-Patterns

- hidden global tool registries
- prompt text that describes unavailable tools
- skills that depend on undeclared files or local paths
- tools whose names or schemas depend on one harness
- policy decisions that are not recorded
- query execution hidden behind broad network access
- broad sandbox network access as a substitute for tool design
- treating built-in harness tools as transactional without protocol support
