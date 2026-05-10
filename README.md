# The Many Habits of Highly Effective Agents

Guiding principles for agent systems that use coupled model-harness substrates
and own the integration layer around them.

This repository is a standalone set of design principles, not an implementation,
API reference, product roadmap, or documentation set for a single project. It
is meant to guide future agent definition libraries in any language. Local
execution matters, but the architecture optimizes for production agents running
in remote sandboxes backed by persistent workspaces.

## Thesis

Effective agents are built from disciplined integration, not from a bare model
call, clever prompt, or generic orchestration loop.

The rented capability is the coupled execution substrate: model, harness loop,
native tools, editing semantics, approval modes, transcript shape, filesystem
assumptions, provider integration, scheduling, retries, and recovery. The model
and harness are trained, post-trained, evaluated, and productized together.
Treating the harness as a thin wrapper loses the behavior being rented.

The durable work is the layer around that substrate:

- definitions that make prompts, tools, policies, state, resources, and output
  contracts explicit
- drivers that bind application intent to harness-native execution
- integration points that expose application systems as typed tools
- skills that package operational knowledge in harness-usable form
- workspace protocols that make remote filesystem state durable
- observability, evals, and conformance suites that make behavior inspectable
  across runs and implementations

An agent system has three strata.

- The **definition layer** describes the agent: prompt, tools, policies, state,
  resources, and outputs.
- The **integration layer** binds definitions to a concrete substrate, sandbox,
  workspace, tools, skills, and event stream.
- The **execution substrate** runs the coupled model-harness loop.

Application teams own the definition and integration layers: definitions,
drivers, tool bridges, skills, tests, evals, conformance, and operational
evidence. Harness and sandbox platforms own the execution substrate. The
boundary between them must preserve work identity, bridge tools without ambient
egress, expose a durable filesystem, stream observable events, and reconnect
without duplicating work.

## Design Map

These files form one contract.

- [PRINCIPLES.md](PRINCIPLES.md) states the constraints that govern the
  repository.
- [DEFINITION.md](DEFINITION.md) defines the portable definition model.
- [CAPABILITIES.md](CAPABILITIES.md) covers tools, policies, feedback,
  completion checking, and egress boundaries.
- [SKILLS.md](SKILLS.md) treats skills as versioned integration assets.
- [STATE.md](STATE.md) covers session state, resources, typed contracts, and
  transaction limits.
- [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md) describes the production topology:
  orchestrator, sandbox control plane, harness runtime, and protocol boundary.
- [WORKSPACES.md](WORKSPACES.md) defines the remote filesystem model.
- [DURABLE-WORK.md](DURABLE-WORK.md) defines work identity, idempotency,
  reconnect, conflicts, and lifecycle separation.
- [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) covers drivers and integration
  points.
- [OBSERVABILITY.md](OBSERVABILITY.md) defines events, transcripts, snapshots,
  traces, and debug bundles.
- [EVALUATION.md](EVALUATION.md) covers agent loops, eval loops, datasets,
  prompt overrides, and conformance tests.
- [GLOSSARY.md](GLOSSARY.md) names the core concepts.

## Scope

This repository sits between application code and any one execution substrate.
It asks:

- What must application teams own when model and harness are coupled?
- What belongs in an agent definition, and what belongs in a driver?
- Which integration points and skills create durable leverage?
- How does a client name work so retries and reconnects are safe?
- What does a remote persistent workspace need to provide?
- Where can transaction guarantees honestly hold?
- How are tool calls, native tool events, and filesystem changes observed?
- What must a driver prove before it conforms?

## Non-Goals

- No full wire protocol.
- No prescribed programming language.
- No preferred model provider, harness, or sandbox platform.
- No harness-independent planning loop.
- No requirement that every implementation expose every feature on day one.
- Local execution remains valuable. Local mode is a special case, not the
  architecture to optimize around.

## Working Rule

When a future implementation has to choose between local convenience and remote
sandbox correctness, remote correctness wins. Local mode is a degenerate case
of the same protocol semantics, not a separate architecture.
