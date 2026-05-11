# Guiding Principles for Unattended Agents that Age Well

This repository is a standalone set of design principles for agent
definition libraries. It is not an implementation, an API reference, a
product roadmap, or the documentation for any one project.

The guide is written for a specific class of system: unattended background
agents that do analytical work. These agents read repositories, write code,
run queries, inspect data, use remote compute, produce durable outputs, and
leave evidence that other people can review. Picture a weekly cohort report
that the agent builds on its own — pulling from a warehouse, writing
intermediate files into a persistent workspace, updating a dashboard, and
leaving enough records that a human can later defend every number. That is
the shape of work this repository has in mind.

## What "Ages Well" Means

"Ages well" is a specific claim, not a mood. It means the agent keeps
producing defensible work as the pieces around it move: model upgrades,
harness version drift, skill and metric changes, data source evolution, and
deployment moves. The goal is portability across those upgrades without
silent behavior shifts.

Every principle and chapter in this repository exists to support that
property. A design that does not survive a model upgrade, a harness change,
or a metric redefinition is not ageing well, regardless of how clean the
code looks on the day it ships.

## Audience and Threat Model

This repository is written for **platform teams serving product teams in a
multi-tenant deployment**. The implicit environment has partially trusted
inputs, multiple tenants sharing infrastructure, data boundaries that
matter, and reviewers who expect to defend analytical outputs later.

Several principles — remote-first sandboxing, egress control, named tools
in place of broad network access, stable public work identifiers, and
records discipline — follow from this threat model rather than from any
universal claim. A single-tenant deployment inside a trusted VPC against a
trusted warehouse may reasonably weaken some of these rules; the corpus
does not, because it is written for the harder case.

## Core Idea

For this class of agent, the useful system is not a bare model call and not
a generic loop. It is five parts working together:

1. **Application intent** — what the product wants, who it is for, and the
   gates around the work.
2. **Agent definition** — what the agent should do, what it may use, and
   what output is expected.
3. **Model used through a harness** — the model plus the harness that gives
   it tools, edit behavior, shell access, skill loading, approvals,
   transcripts, and recovery.
4. **Sandbox with workspace** — the remote runtime and the filesystem where
   work actually happens.
5. **Run records** — the evidence that explains the result after the run is
   over.

The model and the harness should be treated as one working unit. The
harness gives the model its tool shape, edit behavior, shell access, skill
loading, approvals, transcripts, file assumptions, retries, and recovery.
Treating the harness as a thin wrapper around a model API throws away most
of what made the agent useful in the first place.

Application teams should not spend their time rebuilding that harness
loop. The durable work is to define the job, connect application systems
safely, connect that definition to a real harness and sandbox, and keep
enough records to trust the outcome later.

## Concrete Anchors

Codex with OpenAI coding models, and Claude Code or the Claude Agent SDK
with Anthropic models, are useful examples. In both cases the model
matters, but so does the surrounding harness: file search, patch
application, shell execution, query execution, skill loading, approval
flow, transcripts, and recovery. A raw chat loop with the same model is a
different system with different tools and different evidence after the run
is over.

Cloudflare Sandboxes are a useful runtime example. They provide filesystem
isolation, dedicated compute, lifecycle operations, command execution, file
operations, and network policy. The sandbox gives you a runtime; the agent
system still has to supply explicit behavior around work identity,
reconnect, tool routing, persistent workspace state, data access, egress
control, and run records.

## The Centerline

The agent definition sits in the middle of the system.

On one side is the application: user intent, tenant context, authorization,
domain data, metric definitions, query engines, tool handlers, output
consumers, and the gates around unattended work.

On the other side is the runtime: the harness, harness adapter, sandbox,
workspace, skills staged into the runtime, event stream, reconnect
behavior, and records of what happened.

The definition is the stable center. It says what the agent should do,
what it may use, which rules apply, what output is expected, and which
data or runtime requirements affect behavior. It is not the application,
and it is not the harness.

## Layers

These are layers of responsibility. They do not have to be separate
packages or processes.

**1. Application layer.** The application owns product and analytical
context: user, tenant, authorization rules, data sources, metric
definitions, business actions, review rules, and delivery targets. It
exposes that world through named tools, resources, policies, and output
sinks — not by giving the sandbox broad access to application systems.

**2. Application-facing integration.** This direction turns application
systems into definition inputs. It binds tool handlers, query engines,
data access, policies, resources, output sinks, eval fixtures, and review
gates to the definition.

**3. Agent definition.** The definition owns prompt structure, declared
tools, policies, feedback, completion checks, required skills, output
shape, and the parts of state that are meaningful to the agent.

**4. Runtime-facing integration.** This direction connects the definition
to a real harness and sandbox. Its main implementation piece is the
harness adapter: code that runs the definition in one harness while
preserving that harness's behavior.

**5. Sandboxed runtime and workspace.** The harness usually runs in a
remote sandbox. The workspace is the durable filesystem mounted near that
harness. It is not a temporary directory on the caller machine.

**6. Support surfaces.** State, run records, evals, and contract tests
support the layers above. State records what the library needs to
remember. Run records explain what happened. Evals compare behavior.
Contract tests verify integration behavior.

## Do Not Mix These Up

- The **application** owns product context, data meaning, authorization,
  and review gates.
- The **agent definition** says what the agent should do and what it may
  use.
- A **tool** performs an application or data-system action.
- A **skill** teaches the harness how to operate in a repository,
  workflow, or data domain.
- A **harness adapter** translates between the definition and one harness.
- A **sandbox protocol** manages remote runtime, workspace, reconnect, and
  lifecycle.
- A **run record** explains what happened after the run is over.

Keeping these concepts separate is what lets the same ideas survive
different languages, harnesses, sandboxes, and deployment models.

## Reading Order

Start with [PRINCIPLES.md](PRINCIPLES.md) for the short rules.

Use [DEFINITION.md](DEFINITION.md), [CAPABILITIES.md](CAPABILITIES.md),
and [SKILLS.md](SKILLS.md) for the middle of the system: definitions,
tools, data meaning, policies, feedback, completion checks, skills, and
resource requirements.

Use [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) for the two integration
directions: application-facing integration into the definition, and
runtime-facing integration out to the harness and sandbox.

Use [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md),
[WORKSPACES.md](WORKSPACES.md), and [DURABLE-WORK.md](DURABLE-WORK.md) for
the runtime environment: sandboxed execution, persistent workspaces, safe
retries, reconnect, conflicts, and cleanup.

Use [STATE.md](STATE.md), [RUN-RECORDS.md](RUN-RECORDS.md), and
[EVALUATION.md](EVALUATION.md) for support surfaces: state, transaction
limits, records, transcripts, debug bundles, evals, and contract tests.

[GLOSSARY.md](GLOSSARY.md) defines the few terms that need stable names.

## Failure Modes

These principles are meant to prevent common mistakes:

- A retry starts the same turn twice.
- A reconnect creates new work instead of attaching to old work.
- A sandbox cannot see local files that the definition assumed were
  available.
- A tool reaches an application service through broad sandbox network
  access instead of a named, tested path.
- A query runs against stale, wrong, or unauthorized data.
- A metric name hides changed business meaning, filters, or freshness
  rules.
- An expensive or sensitive analysis runs without the right access, cost,
  or review gate.
- A polished answer survives after the query IDs, intermediate files,
  generated outputs, and validation checks are gone.
- A built-in harness file edit is treated as rollbackable even though the
  sandbox never took a snapshot.
- An eval result cannot be explained because the prompt, skills, harness
  version, workspace seed, data lineage, or event stream was not recorded.

## Scope

This repository asks what application teams should own, what belongs in an
agent definition, what belongs in the runtime, which gates surround
unattended work, and what evidence an analytical run needs to leave
behind.

It does not define a full wire protocol, choose a programming language,
prefer a model provider, or define a harness-independent planning loop.

## Working Rule

When local convenience conflicts with remote sandbox correctness, remote
correctness wins.
