# Guiding Principles for Highly Effective Agents that Age Well

This repository is a standalone set of design principles for helping future
agent definition libraries age well. It is not an implementation, API reference,
product roadmap, or docs for one project.

The guide is written for unattended background agents that do complex analytical
work: reading repositories, writing code, running queries, using remote compute,
producing durable outputs, and leaving enough evidence for others to trust or
debug the result.

## Core Idea

Unattended analytical agents are not just model calls, clever prompts, or
generic loops. They work when application intent, agent definition, runtime
environment, and evidence fit together.

For the systems this repository targets, the model and harness are one stack.
The model is used through a harness loop, built-in tools, editing behavior,
approvals, transcripts, filesystem assumptions, provider integration,
scheduling, retries, and recovery. Treating the harness as a thin wrapper around
a model API loses much of the behavior that makes the agent useful.

Application teams should not spend most of their time rebuilding that harness
loop. The durable work is around the harness: connecting analytical intent to a
clear agent definition, then connecting that definition to a real harness,
sandbox, workspace, and records of the work.

## Concrete Anchors

Codex with OpenAI coding models, and Claude Code or the Claude Agent SDK with
Anthropic models, are useful examples of model-harness stacks. In both cases,
the model matters, but so does the shape of the tools around it: file search,
file edit, patch application, shell execution, query execution, skill loading,
approvals, transcripts, and recovery.

A raw chat loop with the same model is a different system. It may still be
useful, but it has different tools, different failure modes, and different
evidence after the run is over.

Cloudflare Sandboxes are a useful example of the runtime side: isolated
filesystem, dedicated containerized compute, lifecycle API, command execution,
file operations, and network policy. A production agent system still needs
explicit protocol behavior around that sandbox for durable work, persistent
workspace state, reconnect, data access, network controls, and run records.

## The Centerline

The agent definition sits in the middle.

On one side is the application layer: user intent, domain data, query engines,
authorization, tool handlers, output consumers, and business context.

On the other side is the runtime environment: the harness, sandbox, workspace,
and records of what happened.

The definition is the stable center. It says what the agent is supposed to do,
what it may use, which rules apply, and what output is expected. It should not
depend on private details of either side.

## Layers of Responsibility

These are layers of responsibility, not necessarily separate processes or
packages.

**1. Application layer**

The application layer owns product and analytical context. It knows the user,
tenant, authorization rules, data sources, metric definitions, business actions,
and where the final output goes.

It should expose that world through clear tools, policies, and resources, not by
letting the sandbox reach broadly into application systems.

**2. Application-facing integration**

This direction turns application systems into definition inputs. It binds tool
handlers, data access, policies, resources, output sinks, and eval fixtures to
the agent definition.

This is where product constraints enter the agent system.

**3. Agent definition**

The definition says what the agent is supposed to do. It owns prompt structure,
declared tools, policies, feedback, completion checks, resource requirements,
output shape, and the parts of state that are meaningful to the agent.

The definition is not the application. It is also not the harness. It is the
contract between them.

**4. Runtime-facing integration**

This direction connects the definition to a real harness and sandbox. Its main
implementation piece is the harness adapter: code that runs the definition in
one harness without making the definition depend on that harness.

Runtime-facing integration owns the practical work of staging skills, routing
tools, mapping events, and talking to the sandbox protocol.

**5. Sandboxed runtime and workspace**

In production, the harness usually runs in a remote sandbox. The workspace is
the durable filesystem mounted near that harness. It is not a temporary
directory on the caller's machine.

The sandbox protocol owns the runtime lifecycle. Reconnect, cleanup, and retry
behavior need explicit rules.

**6. Support surfaces**

State, run records, evals, and contract tests support the layers above. They are
not another place to put application intent or harness behavior.

State records what the library needs to remember. Run records explain what
happened. Evals compare behavior. Contract tests verify that an integration
layer follows shared expectations.

## Do Not Mix These Up

- A **definition** says what the agent should do.
- A **tool** performs an application or data-system action.
- A **skill** teaches the harness how to operate in a repository, workflow, or
  data domain.
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
data access, policies, feedback, completion checks, skills, and resource
requirements.

Use [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) for the two integration
directions: application-facing integration into the definition, and
runtime-facing integration out to the harness and sandbox.

Use [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md), [WORKSPACES.md](WORKSPACES.md),
and [DURABLE-WORK.md](DURABLE-WORK.md) for the runtime environment: sandboxed
execution, persistent workspaces, safe retries, reconnect, conflicts, and
cleanup.

Use [STATE.md](STATE.md), [RUN-RECORDS.md](RUN-RECORDS.md), and
[EVALUATION.md](EVALUATION.md) for support surfaces: state, transaction limits,
records, transcripts, debug bundles, evals, and contract tests.

[GLOSSARY.md](GLOSSARY.md) defines the few terms that need stable names.

## Failure Modes

These principles are meant to prevent common mistakes:

- A retry starts the same turn twice because the request had no stable
  idempotency key.
- A reconnect creates new work instead of attaching to the old work.
- A sandbox cannot see local files that the definition assumed were available.
- A tool call reaches an application service through broad sandbox network
  access instead of a named, tested tool.
- A query runs against stale, wrong, or unauthorized data and the run record does
  not show what happened.
- A polished answer survives after the query IDs, intermediate files, generated
  outputs, and validation checks are gone.
- A built-in harness file edit is treated as rollbackable even though the
  sandbox never took a snapshot.
- An eval result cannot be explained because the rendered prompt, skills,
  harness version, workspace seed, data lineage, or event stream was not
  recorded.

## Scope

This repository asks what application teams should own, what belongs in an agent
definition, what belongs in the runtime, and what evidence an unattended
analytical run needs to leave behind.

It does not define a full wire protocol, choose a programming language, prefer a
model provider, or define a harness-independent planning loop.

## Working Rule

When local convenience conflicts with remote sandbox correctness, remote
correctness wins.
