# Glossary

**Agent definition.** Typed artifact containing prompt structure, tools,
policies, feedback, completion checks, state, resources, and output contracts.

**Application egress.** External access to application services or data.
Prefer declared tools over ambient sandbox network access.

**Caller-owned identifier.** Stable name supplied by the orchestrator or
application for durable work.

**Canonical event.** Normalized event emitted by a driver so different
harnesses can be observed uniformly.

**Completion check.** Verification step that decides whether a candidate final
response satisfies the definition's done criteria.

**Compute lifecycle.** Start, sleep, restart, and stop behavior of sandbox or
harness compute.

**Connection lifecycle.** Transport connection behavior between driver,
control plane, and harness.

**Control plane.** Service boundary that owns sandbox lifecycle, routing,
auth, workspace mounting, durable ledgers, and reconnect semantics.

**Definition tool.** Tool declared by the agent definition and fulfilled by the
orchestrator or application.

**Driver.** Code and configuration that bind application-owned definitions,
tools, skills, and work identifiers to a concrete model-harness substrate and
sandbox protocol.

**Durable work.** Caller-named work that survives transient disconnects,
retries, and compute restarts.

**Feedback.** Advisory message emitted after an event to steer future behavior
without blocking the event.

**Harness.** Runtime that owns the model act loop, native tools, provider
integration, and sandbox-local execution.

**Model-harness substrate.** Coupled execution product made from a model,
harness loop, native tools, message protocol, filesystem assumptions, and
runtime policies.

**Native harness tool.** Capability provided by the harness, such as shell,
file edit, search, or patch. Its semantics are harness-specific unless wrapped
by the driver.

**Policy.** Fail-closed invariant checked at a defined boundary.

**Progressive disclosure.** Rendering strategy where details and capabilities
become visible only when needed.

**Remote sandbox.** Isolated runtime environment that hosts the harness and its
workspace outside the application process.

**Resource.** Injected dependency with explicit lifetime, such as filesystem,
clock, object store, or database client.

**Semantic contract.** Hashable set of inputs and declarations that affect a
unit of work's behavior.

**Session.** Durable state container associated with one or more units of work.

**Skill.** Versioned integration asset that packages operating knowledge,
examples, scripts, schemas, or runtime assets for a harness to use.

**Snapshotable resource.** Resource that can capture and restore state through
a transaction boundary.

**Structured output.** Typed output contract for final agent results.

**Tool result.** Typed success or failure returned to the agent after a tool
call.

**Transaction.** Scoped atomicity guarantee over framework-owned state and
snapshotable resources.

**Transport error.** Failure to exchange messages reliably. Distinct from a
tool failure, which is an application-level result.

**Turn.** One caller-named unit of interaction with a thread or session.

**Workspace.** Remote persistent filesystem mounted near the harness and used
as the agent's data plane.

**Work lifecycle.** Creation, progress, terminal result, retention, and
deletion of caller-named work.
