# Guiding Principles for Unattended Agents that Age Well

This repository is a standalone set of design principles for agent definition
libraries. It is not an implementation, API reference, product roadmap, or docs
for one project.

The guide is written for unattended background agents that do analytical work:
reading repositories, writing code, running queries, inspecting data, using
remote compute, producing durable outputs, and leaving evidence that other
people can review.

## Core Idea

For this class of agent, the useful system is not a bare model call and not a
generic loop. It is the application intent, the agent definition, the model as
used through a harness, the sandbox and workspace where work happens, and the
records that explain the result.

The model and harness should be treated as one working unit. The harness gives
the model its tools, edit behavior, shell access, skill loading, approvals,
transcripts, file assumptions, retries, and recovery behavior. Treating the
harness as a thin wrapper around a model API loses much of what makes the agent
useful.

Application teams should not spend most of their time rebuilding that harness
loop. The durable work is to define the job, connect application systems into it
safely, connect that definition to a real harness and sandbox, and keep enough
records to trust the outcome later.

## Concrete Anchors

Codex with OpenAI coding models, and Claude Code or the Claude Agent SDK with
Anthropic models, are useful examples. In both cases the model matters, but so
does the surrounding harness: file search, patch application, shell execution,
query execution, skill loading, approval flow, transcripts, and recovery.

A raw chat loop with the same model is a different system. It may still be
useful, but it has different tools, different failure modes, and different
evidence after the run is over.

Cloudflare Sandboxes are a useful runtime example. They provide isolated
filesystem, dedicated compute, lifecycle operations, command execution, file
operations, and network policy. An agent system still needs explicit behavior
around work identity, reconnect, tool routing, persistent workspace state, data
access, egress control, and run records.

## The Centerline

The agent definition sits in the middle.

On one side is the application: user intent, tenant context, authorization,
domain data, metric definitions, query engines, tool handlers, output
consumers, and the gates around unattended work.

On the other side is the runtime: the harness, harness adapter, sandbox,
workspace, skills staged into the runtime, event stream, reconnect behavior, and
records of what happened.

The definition is the stable center. It says what the agent should do, what it
may use, which rules apply, what output is expected, and which data or runtime
requirements affect behavior. It is not the application, and it is not the
harness.

## Layers

These are layers of responsibility. They do not have to be separate packages or
processes.

**1. Application layer**

The application owns product and analytical context. It knows the user, tenant,
authorization rules, data sources, metric definitions, business actions, review
rules, and delivery targets.

It should expose that world through named tools, resources, policies, and output
sinks, not by giving the sandbox broad access to application systems.

**2. Application-facing integration**

This direction turns application systems into definition inputs. It binds tool
handlers, query engines, data access, policies, resources, output sinks, eval
fixtures, and review gates to the definition.

**3. Agent definition**

The definition owns prompt structure, declared tools, policies, feedback,
completion checks, required skills, output shape, and the parts of state that
are meaningful to the agent.

**4. Runtime-facing integration**

This direction connects the definition to a real harness and sandbox. Its main
implementation piece is the harness adapter: code that runs the definition in
one harness while preserving that harness's behavior.

**5. Sandboxed runtime and workspace**

The harness usually runs in a remote sandbox. The workspace is the durable
filesystem mounted near that harness. It is not a temporary directory on the
caller machine.

**6. Support surfaces**

State, run records, evals, and contract tests support the layers above. State
records what the library needs to remember. Run records explain what happened.
Evals compare behavior. Contract tests verify integration behavior.

## Do Not Mix These Up

- The application owns product context, data meaning, authorization, and review
  gates.
- The definition says what the agent should do and what it may use.
- A tool performs an application or data-system action.
- A skill teaches the harness how to operate in a repository, workflow, or data
  domain.
- A harness adapter translates between the definition and one harness.
- A sandbox protocol manages remote runtime, workspace, reconnect, and
  lifecycle.
- A run record explains what happened after the run is over.

Keeping these concepts separate is what lets the same ideas survive different
languages, harnesses, sandboxes, and deployment models.

## What This Repository Covers

Start with [PRINCIPLES.md](PRINCIPLES.md) for the short rules.

Use [DEFINITION.md](DEFINITION.md), [CAPABILITIES.md](CAPABILITIES.md), and
[SKILLS.md](SKILLS.md) for the middle of the system: definitions, tools, data
meaning, policies, feedback, completion checks, skills, and resource
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

- A retry starts the same turn twice.
- A reconnect creates new work instead of attaching to old work.
- A sandbox cannot see local files that the definition assumed were available.
- A tool reaches an application service through broad sandbox network access
  instead of a named, tested path.
- A query runs against stale, wrong, or unauthorized data.
- A metric name hides changed business meaning, filters, or freshness rules.
- An expensive or sensitive analysis runs without the right access, cost, or
  review gate.
- A polished answer survives after the query IDs, intermediate files, generated
  outputs, and validation checks are gone.
- A built-in harness file edit is treated as rollbackable even though the
  sandbox never took a snapshot.
- An eval result cannot be explained because the prompt, skills, harness
  version, workspace seed, data lineage, or event stream was not recorded.

## Scope

This repository asks what application teams should own, what belongs in an
agent definition, what belongs in the runtime, which gates surround unattended
work, and what evidence an analytical run needs to leave behind.

It does not define a full wire protocol, choose a programming language, prefer a
model provider, or define a harness-independent planning loop.

## Working Rule

When local convenience conflicts with remote sandbox correctness, remote
correctness wins.
