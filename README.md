# Guiding Principles for Highly Effective Agents that Age Well

This repository is a standalone set of design principles for agent systems that
need to work across model-harness stacks, remote sandboxes, and persistent
workspaces. It is not an implementation, API reference, product roadmap, or docs
for one project.

The goal is to help future agent definition libraries age well. Local execution
matters, but production agents usually run somewhere else: inside a sandbox,
near a persistent workspace, behind a protocol, with a harness that already has
its own tools and recovery behavior.

## Thesis

Effective agents are not just model calls, clever prompts, or generic loops.
They work when the model, harness, tools, workspace, skills, tests, and run
records fit together.

The model and harness are one stack. The model is trained, evaluated, and
shipped with a harness loop, built-in tools, editing behavior, approvals,
transcripts, filesystem assumptions, provider integration, scheduling, retries,
and recovery. If a library treats the harness as a thin wrapper around a model
API, it loses much of the behavior that makes the agent useful.

The long-term work for application teams is the layer around that stack:
definitions, harness adapters, tool bridges, skills, workspace protocols, logs,
traces, evals, and contract tests. That is where application intent meets the
harness without pretending every harness behaves the same way.

The mental model is simple:

- The **definition layer** says what the agent is supposed to do.
- The **integration layer** connects that definition to a real harness,
  sandbox, workspace, tools, skills, and event stream.
- The **model-harness runtime** runs the model loop and built-in harness tools.

Application teams own the definition and integration layers. Harness and
sandbox platforms own the runtime. The line between them has to do practical
work: preserve work identity, route tool calls without broad network access,
expose a durable filesystem, stream useful events, and reconnect without
starting duplicate work.

## Failure Modes

These principles are meant to prevent common mistakes:

- A retry starts the same turn twice because the caller did not name the work.
- A reconnect creates new work instead of attaching to the old work.
- A sandbox cannot see local files that the definition assumed were available.
- A tool call reaches an application service through broad sandbox network
  access instead of a named, tested tool.
- A built-in harness file edit is treated as rollbackable even though the
  sandbox never took a snapshot.
- An eval result cannot be explained because the rendered prompt, skills,
  harness version, workspace seed, or event stream was not recorded.

## Design Map

Read the files in this order:

- [PRINCIPLES.md](PRINCIPLES.md) gives the short rules.
- [DEFINITION.md](DEFINITION.md) describes what belongs in an agent definition.
- [CAPABILITIES.md](CAPABILITIES.md) explains tools, policies, feedback,
  completion checks, and external access.
- [SKILLS.md](SKILLS.md) describes skills as versioned operating knowledge.
- [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md), [WORKSPACES.md](WORKSPACES.md),
  and [DURABLE-WORK.md](DURABLE-WORK.md) explain the runtime shape: sandbox,
  workspace, work identity, reconnect, conflicts, and cleanup.
- [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) explains harness adapters and
  the bridges around them.
- [STATE.md](STATE.md) explains event-driven state, resources, transactions,
  and idempotency.
- [RUN-RECORDS.md](RUN-RECORDS.md) explains the records needed to understand a
  run later.
- [EVALUATION.md](EVALUATION.md) explains eval loops, datasets, prompt
  overrides, and contract tests.
- [GLOSSARY.md](GLOSSARY.md) gives stable names for the few concepts worth
  naming.

## Scope

This repository sits between application code and any one model-harness runtime.
It asks what application teams should own, what belongs in an agent definition,
what belongs in the integration layer, how remote work is named, how tool calls
are routed, where rollback can honestly hold, and what evidence every run needs
to leave behind.

It does not define a full wire protocol, choose a programming language, prefer a
model provider, or define a harness-independent planning loop. Local mode is
important, but it should be the same design in a simpler setting, not a second
design.

## Working Rule

When local convenience conflicts with remote sandbox correctness, remote
correctness wins.
