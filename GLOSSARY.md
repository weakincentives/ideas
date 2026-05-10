# Glossary

**Agent definition.** Typed record containing prompt structure, tools, policies,
feedback, completion checks, state, resources, and output shape.

**Application external access.** Access to application services or data. Prefer
declared tools over broad sandbox network access.

**Built-in harness tool.** Tool provided by the harness, such as shell, file
edit, search, or patch. Its behavior is harness-specific unless wrapped by the
harness adapter.

**Caller-owned identifier.** Stable name supplied by the orchestrator or
application for durable work.

**Completion check.** Step that decides whether a candidate final response
satisfies the definition's done criteria.

**Compute lifecycle.** Start, sleep, restart, and stop behavior of sandbox or
harness compute.

**Connection lifecycle.** Transport connection behavior between harness adapter,
control plane, and harness.

**Contract test suite.** Tests that prove an integration layer implements the
shared behavior.

**Control plane.** Service that owns sandbox lifecycle, routing, auth, workspace
mounting, durable ledgers, and reconnect behavior.

**Definition tool.** Tool declared by the agent definition and fulfilled by the
orchestrator or application.

**Durable work.** Caller-named work that survives transient disconnects,
retries, and compute restarts.

**Feedback.** Advisory message emitted after an event to steer future behavior
without blocking the event.

**Harness.** Runtime that owns the model act loop, built-in tools, provider
integration, and sandbox-local execution.

**Harness adapter.** Integration-layer code that translates definitions, tools,
skills, and events into one harness's rules and formats.

**Model-harness runtime.** The model plus the harness that runs it: model calls,
harness loop, built-in tools, protocol, filesystem assumptions, approvals, and
recovery behavior.

**Policy.** Fail-closed rule checked at a known point in the run.

**Progressive disclosure.** Showing details and capabilities only when the
agent needs them.

**Remote sandbox.** Isolated runtime environment that runs the harness and its
workspace outside the application process.

**Resource.** Injected dependency with explicit lifetime, such as filesystem,
clock, object store, or database client.

**Run record.** Event, transcript, trace, debug bundle, or related record that
helps reconstruct what happened during a run.

**Session.** Durable state container associated with one or more units of work.

**Skill.** Versioned package of operating knowledge, examples, scripts, schemas,
or runtime files for a harness to use.

**Snapshotable resource.** Resource that can capture and restore its state
during a transaction.

**Standard event.** Event emitted by a harness adapter so different harnesses
can be observed in the same way.

**Structured output.** Typed shape for final agent results.

**Tool result.** Typed success or failure returned to the agent after a tool
call.

**Transaction.** Scoped guarantee over library-owned state and resources that
support snapshots.

**Transport error.** Failure to exchange messages reliably. Distinct from a
tool failure, which is an application-level result.

**Turn.** One caller-named unit of interaction with a thread or session.

**Work contract.** Hashable set of inputs and declarations that affect a unit of
work's behavior.

**Work lifecycle.** Creation, progress, terminal result, retention, and deletion
of caller-named work.

**Workspace.** Remote persistent filesystem mounted near the harness and used by
the agent.
