# Capabilities

Capabilities are the declared ways an agent can act. Application side effects
are named tools, not hidden access.

## Tool Classes

There are two tool classes.

**Definition tools** are declared by the agent definition and fulfilled by the
orchestrator or application. The definition owns their schema, policy, result
type, and documentation.

**Built-in harness tools** are provided by the harness inside the sandbox.
Common examples include shell, file edit, file read, search, and patch tools.
They are not definition tools unless the harness adapter deliberately wraps them
behind a shared tool shape.

The distinction matters for transactions and records. A definition library can
wrap definition tools. It can only observe built-in harness tools unless the
sandbox protocol exposes control points for policy, snapshot, and rollback.

## Skills

Skills sit next to capabilities. A skill packages operating knowledge that helps
the model-harness runtime use a repository, tool, workflow, or domain correctly.

Skills may include instructions, examples, scripts, manifests, schemas,
fixtures, or references to definition tools. They are versioned and tested with
the harness adapter because harnesses differ in how they discover, mount, and
use skills.

Teams put harness-facing operating knowledge in skills. Skills do not replace
typed tools, and typed tools do not replace skills.

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

## Policies

Policies are fail-closed rules. They can run before render, before a tool call,
after a tool call, before final output, or during reconnect.

Good policies are small, typed, predictable, and explainable. They produce
denial records that show up in traces and debug bundles.

## Feedback

Feedback is advisory. It nudges the agent after an event without blocking the
event. Feedback covers style, sequencing hints, read-before-write reminders,
and context steering.

Policies prevent invalid behavior. Feedback improves behavior that is still
allowed.

## Completion Checks

Completion checks verify that "done" really means done. They run after the
harness produces a candidate final response and may inspect the session,
workspace metadata, tool history, or structured output.

Completion checks are predictable when possible. LLM-as-judge can help, but it
is an evaluator with its own model, threshold, prompt hash, and trace.

## External Access

Remote sandboxes restrict application network access. Model-provider traffic may
be allowed for the harness. Application traffic flows through definition tools
fulfilled by the orchestrator or through explicitly declared sandbox network
profiles.

This gives reviewers a clear record:

- every application action has a tool name
- every tool call has typed input and output
- every call carries caller-owned work identifiers
- every denial or failure is visible

## Anti-Patterns

- hidden global tool registries
- prompt text that describes unavailable tools
- skills that depend on undeclared files or local paths
- tools whose names or schemas depend on one harness
- policy decisions that are not recorded
- broad sandbox network access as a substitute for tool design
- treating built-in harness tools as transactional without protocol support
