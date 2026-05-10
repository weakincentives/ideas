# Glossary

**Agent definition.** Typed record containing prompt structure, tools,
policies, feedback, completion checks, state, resources, and output contracts.

**Application external access.** Access to application services or data. Prefer
declared tools over broad sandbox network access.

**Caller-owned identifier.** Stable name supplied by the orchestrator or
application for durable work.

**Standard event.** Normalized event emitted by a harness adapter so different
harnesses can be observed in the same way.

**Completion check.** Verification step that decides whether a candidate final
response satisfies the definition's done criteria.

**Compute lifecycle.** Start, sleep, restart, and stop behavior of sandbox or
harness compute.

**Connection lifecycle.** Transport connection behavior between harness adapter,
control plane, and harness.

**Control plane.** Service that owns sandbox lifecycle, routing, auth, workspace
mounting, durable ledgers, and reconnect behavior.

**Definition tool.** Tool declared by the agent definition and fulfilled by the
orchestrator or application.

**Model-harness runtime.** The model plus the harness that runs it: model
calls, harness loop, built-in tools, protocol, filesystem assumptions, approval
modes, and recovery behavior.

**Durable work.** Caller-named work that survives transient disconnects,
retries, and compute restarts.

**Feedback.** Advisory message emitted after an event to steer future behavior
without blocking the event.

**Harness.** Runtime that owns the model act loop, built-in tools, provider
integration, and sandbox-local execution.

**Harness adapter.** Integration-layer component that translates portable
definitions, tools, skills, and events into one harness's rules and formats.

**Built-in harness tool.** Capability provided by the harness, such as shell,
file edit, search, or patch. Its behavior is harness-specific unless wrapped by
the harness adapter.

**Policy.** Fail-closed rule checked at a known point in the run.

**Progressive disclosure.** Showing details and capabilities only when the
agent needs them.

**Remote sandbox.** Isolated runtime environment that runs the harness and its
workspace outside the application process.

**Resource.** Injected dependency with explicit lifetime, such as filesystem,
clock, object store, or database client.

**Work contract.** Hashable set of inputs and declarations that affect a unit of
work's behavior.

**Session.** Durable state container associated with one or more units of work.

**Skill.** Versioned integration asset that packages operating knowledge,
examples, scripts, schemas, or runtime assets for a harness to use.

**Snapshotable resource.** Resource that can capture and restore its state
during a transaction.

**Structured output.** Typed output contract for final agent results.

**Contract test suite.** Test suite that proves an integration layer implements
the shared contract.

**Tool result.** Typed success or failure returned to the agent after a tool
call.

**Transaction.** Scoped guarantee over framework-owned state and resources that
support snapshots.

**Transport error.** Failure to exchange messages reliably. Distinct from a
tool failure, which is an application-level result.

**Turn.** One caller-named unit of interaction with a thread or session.

**Workspace.** Remote persistent filesystem mounted near the harness and used
as the agent's data plane.

**Work lifecycle.** Creation, progress, terminal result, retention, and
deletion of caller-named work.
