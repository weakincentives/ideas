# Glossary

**Agent definition.** Typed record in the middle of the system. It says what the
agent should do, what it may use, which rules apply, and what output is
expected.

**Application-facing integration.** Code that binds application systems into the
agent definition: tools, policies, resources, output consumers, eval fixtures,
and caller-owned work names.

**Built-in harness tool.** Tool provided by the harness, such as shell, file
edit, search, or patch. Its behavior is harness-specific unless wrapped by the
harness adapter.

**Definition tool.** Tool declared by the agent definition and fulfilled through
application-facing integration.

**Durable work.** Caller-named work that survives transient disconnects,
retries, and compute restarts.

**Harness adapter.** Runtime-facing integration code that translates a
definition into one harness's rules and formats.

**Model-harness runtime.** The model plus the harness that runs it: model calls,
harness loop, built-in tools, protocol, filesystem assumptions, approvals, and
recovery behavior.

**Remote sandbox.** Isolated runtime environment that runs the harness and its
workspace outside the application process.

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
