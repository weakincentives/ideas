# Glossary

**Agent definition.** Typed record in the middle of the system. It says what the
agent should do, what it may use, which rules apply, and what output is
expected.

**Analytical agent.** Unattended background agent that reads data or
repositories, writes code, runs queries, produces durable outputs, and leaves
records that make the result reviewable.

**Analytical job.** Query, code execution, report generation, dashboard update,
or other long-running analytical action with status, outputs, records, and
possibly cancellation.

**Application-facing integration.** Code that binds application systems into the
agent definition: tools, query engines, policies, resources, output consumers,
and eval fixtures.

**Built-in harness tool.** Tool provided by the harness, such as shell, file
edit, search, or patch. Its behavior is harness-specific unless wrapped by the
harness adapter.

**Data tool.** Definition tool that reads, queries, writes, or publishes through
an application data system, such as a warehouse, object store, dashboard, or
application database.

**Definition tool.** Tool declared by the agent definition and fulfilled through
application-facing integration.

**Durable work.** Work with stable public identity that survives transient
disconnects, retries, and compute restarts.

**Harness adapter.** Runtime-facing integration code that translates a
definition into one harness's rules and formats.

**Metric definition.** Business meaning, owner, filters, freshness rules, and
data limits that make an analytical result interpretable.

**Model-harness runtime.** The model plus the harness that runs it: model calls,
harness loop, built-in tools, filesystem assumptions, approvals, and recovery
behavior. Examples include Codex with OpenAI coding models, and Claude Code or
the Claude Agent SDK with Anthropic models.

**Remote sandbox.** Isolated runtime environment that runs the harness and its
workspace outside the application process. Cloudflare Sandboxes are one
concrete example.

**Run record.** Event, transcript, trace, debug bundle, or related record that
helps reconstruct what happened during a run.

**Runtime-facing integration.** Code that connects a definition to a harness,
sandbox, workspace, skill format, and event stream.

**Skill.** Versioned package of operating knowledge, examples, scripts, schemas,
or runtime files for a harness to use.

**Standard event.** Event emitted by a harness adapter so different harnesses
can be observed in the same way.

**Work contract.** Hashable set of inputs and declarations that affect a unit of
work's behavior.

**Workspace.** Remote persistent filesystem mounted near the harness and used by
the agent.
