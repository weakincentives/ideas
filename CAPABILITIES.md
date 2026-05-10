# Capabilities

Capabilities are the agent's declared ways to affect the world. The important
rule is simple: application side effects should be named tools, not ambient
access.

## Tool Classes

There are two tool classes.

**Definition tools** are declared by the agent definition and fulfilled by the
orchestrator or application. They are portable across harnesses because the
definition owns their schema, policy, result type, and documentation.

**Native harness tools** are provided by the harness inside the sandbox. Common
examples include shell, file edit, file read, search, and patch tools. They are
not definition tools unless the driver deliberately re-exposes them through a
portable contract.

The distinction matters for transactions and observability. A definition
library can wrap definition tools. It can only observe native tools unless the
sandbox protocol exposes control points for snapshot, policy, and rollback.

## Skills

Skills are capability-adjacent integration assets. A skill packages operational
knowledge that helps the execution substrate use a repository, tool, workflow,
or domain correctly.

Skills may include instructions, examples, scripts, manifests, schemas,
fixtures, or references to definition tools. They should be versioned and tested
with the driver because harnesses differ in how they discover, mount, and use
skills.

Skills are where teams should put harness-facing operating knowledge. They are
not a substitute for typed tools, and typed tools are not a substitute for
skills.

## Tool Contract

A tool should have:

- stable portable name
- typed input schema
- typed result schema
- human-readable description
- examples
- visibility or phase rules
- policy hooks
- timeout and budget behavior
- idempotency guidance for external effects

Portable tool names should fit the lowest common denominator of the target
protocols: lowercase ASCII letters, digits, underscore, and hyphen are usually
safe. Avoid names whose meaning depends on one harness.

## Tool Results

Tool failure is data, not a framework crash. A failed tool should return a typed
error result that the agent can reason about. Transport failure is different: it
means the caller does not know whether the tool request was received or
completed and must reconcile with the sandbox.

## Policies

Policies are fail-closed invariants. They can apply before render, before a
tool call, after a tool call, before final output, or during reconnect.

Good policies are small, typed, deterministic, and explainable. They should
produce denial records that can be inspected in traces and debug bundles.

## Feedback

Feedback is advisory. It nudges the agent after an event without blocking the
event. Feedback is useful for style, sequencing hints, read-before-write
reminders, and context steering.

Policies prevent invalid behavior. Feedback improves behavior that is still
allowed.

## Completion Checking

Completion checks verify that "done" really means done. They run after the
harness produces a candidate final response and may inspect the session,
workspace metadata, tool history, or structured output.

Completion checks should be deterministic when possible. LLM-as-judge can be
useful, but should be treated as a typed evaluator with its own model,
threshold, prompt hash, and trace.

## Egress Boundary

Remote sandboxes should restrict application egress. Model-provider traffic may
be allowed for the harness. Application traffic should flow through definition
tools fulfilled by the orchestrator or through explicitly declared sandbox
egress profiles.

This creates an auditable capability surface:

- every application action has a tool name
- every tool call has typed input and output
- every call carries caller-owned work identifiers
- every denial or failure is observable

## Anti-Patterns

- hidden global tool registries
- prompt text that describes unavailable tools
- skills that depend on undeclared files or ambient local paths
- tools whose names or schemas depend on a single harness
- policy decisions that are not recorded
- broad network access from the sandbox as a substitute for tool design
- treating native harness tools as transactional without a protocol guarantee
