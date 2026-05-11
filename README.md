# Guiding Principles for Highly Effective Agents that Age Well

This repository is a standalone set of design principles for agent systems that
need to work across model-harness stacks, remote sandboxes, and persistent
workspaces. It is not an implementation, API reference, product roadmap, or docs
for one project.

The goal is to help future agent definition libraries age well. Local execution
matters, but production agents usually run somewhere else: inside a sandbox,
near a persistent workspace, behind a protocol, with a harness that already has
its own tools and recovery behavior.

## Core Idea

Effective agents are not just model calls, clever prompts, or generic loops.
They work when the application, definition, harness, sandbox, workspace, skills,
tests, and run records fit together.

The model and harness are one stack. The model is trained, evaluated, and
shipped with a harness loop, built-in tools, editing behavior, approvals,
transcripts, filesystem assumptions, provider integration, scheduling, retries,
and recovery. Treating the harness as a thin wrapper around a model API loses
much of the behavior that makes the agent useful.

Application teams should not spend most of their time rebuilding that harness
loop. The durable work is around the harness: connecting application intent to a
clear agent definition, then connecting that definition to a real harness,
sandbox, workspace, and event stream.

## The Centerline

The agent definition sits in the middle.

On one side is the application layer: product flows, domain data, authorization,
tool handlers, output consumers, user intent, and caller-owned work names.

On the other side is the runtime environment: the harness, sandbox, workspace,
built-in tools, provider traffic, event stream, reconnect behavior, and
retention rules.

The definition is the stable center. It says what the agent is supposed to do,
what it may use, which rules apply, and what output is expected. It should not
depend on private details of either side.

## Layers of Responsibility

These are layers of responsibility, not necessarily separate processes or
packages.

**1. Application layer**

The application layer owns product behavior. It knows the user, tenant,
authorization rules, domain data, business actions, and where the final output
goes.

It should expose that world through clear tools, policies, resources, and work
names, not by letting the sandbox reach broadly into application systems.

**2. Application-facing integration**

This direction turns application systems into definition inputs. It binds tool
handlers, policies, resources, output sinks, eval fixtures, and caller-owned
work names to the agent definition.

This is where product constraints enter the agent system.

**3. Agent definition**

The definition says what the agent is supposed to do. It owns prompt structure,
declared tools, policies, feedback, completion checks, resource requirements,
output shape, and the parts of state that are meaningful to the agent.

The definition is not the application. It is also not the harness. It is the
contract between them.

**4. Runtime-facing integration**

This direction connects the definition to a real harness and sandbox. Its main
implementation piece is the harness adapter: code that translates a shared
definition into one harness's prompt shape, tool shape, skill format, event
stream, approval behavior, and result format.

Runtime-facing integration also stages skills, routes definition tool requests
back to application handlers, maps harness events into run records, and calls
the sandbox protocol for workspace and lifecycle operations.

**5. Sandboxed runtime and workspace**

In production, the harness usually runs in a remote sandbox. The workspace is
the durable filesystem mounted near that harness. It is not a temporary
directory on the caller's machine.

The sandbox protocol owns runtime lifecycle, workspace access, reconnect,
isolation, and cleanup. The caller names work. A reconnect attaches to existing
work. A disconnect does not cancel work by default.

**6. Support surfaces**

State, run records, evals, and contract tests support the layers above. They are
not another place to put application intent or harness behavior.

State records what the library needs to remember. Run records explain what
happened. Evals compare behavior. Contract tests prove that an integration layer
implements the shared expectations.

## Do Not Mix These Up

- A **definition** says what the agent should do.
- A **tool** performs an application action.
- A **skill** teaches the harness how to operate in a repository, workflow, or
  domain.
- A **harness adapter** translates between the definition and one harness.
- A **sandbox protocol** manages remote runtime, workspace, reconnect, and
  lifecycle.
- A **run record** explains what happened after the run is over.

Keeping these concepts separate is what lets the same ideas survive different
languages, harnesses, sandboxes, and deployment models.

## What This Repository Covers

Start with [PRINCIPLES.md](PRINCIPLES.md) for the short rules.

Use [DEFINITION.md](DEFINITION.md), [CAPABILITIES.md](CAPABILITIES.md), and
[SKILLS.md](SKILLS.md) for the middle of the system: the definition, tools,
policies, feedback, completion checks, skills, and resource requirements.

Use [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) for the two integration
directions: application-facing integration into the definition, and
runtime-facing integration out to the harness and sandbox.

Use [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md), [WORKSPACES.md](WORKSPACES.md),
and [DURABLE-WORK.md](DURABLE-WORK.md) for the runtime environment: sandboxed
execution, persistent workspaces, caller-named work, reconnect, conflicts, and
cleanup.

Use [STATE.md](STATE.md), [RUN-RECORDS.md](RUN-RECORDS.md), and
[EVALUATION.md](EVALUATION.md) for support surfaces: state, transaction limits,
records, transcripts, debug bundles, evals, and contract tests.

[GLOSSARY.md](GLOSSARY.md) defines the few terms that need stable names.

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

## Scope

This repository asks what application teams should own, what belongs in an
agent definition, what belongs in each integration direction, how remote work is
named, how tool calls are routed, where rollback can honestly hold, and what
evidence every run needs to leave behind.

It does not define a full wire protocol, choose a programming language, prefer a
model provider, or define a harness-independent planning loop.

## Working Rule

When local convenience conflicts with remote sandbox correctness, remote
correctness wins.
