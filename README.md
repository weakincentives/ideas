# Guiding Principles for Highly Effective Agents that Age Well

Guiding principles for agent systems built around model-harness stacks, remote
sandboxes, durable workspaces, and explicit integration layers.

This repository is a standalone set of design principles, not an implementation,
API reference, product roadmap, or documentation set for a single project. It
is meant to help future agent definition libraries in any language age well.
Local execution matters, but production agents usually run in remote sandboxes
backed by persistent workspaces.

## Thesis

Effective agents are not just model calls, clever prompts, or generic loops.
They work when the model, harness, tools, workspace, skills, tests, and
operational record fit together.

The model and harness are one stack. The model is trained, post-trained,
evaluated, and shipped with a harness loop, built-in tools, editing behavior,
approval modes, transcript shape, filesystem assumptions, provider integration,
scheduling, retries, and recovery. Treating the harness as a thin wrapper loses
the behavior that makes the agent useful.

The durable work is the layer around that stack:

- definitions that make prompts, tools, policies, state, resources, and output
  contracts explicit
- harness adapters that translate application intent into harness-specific
  execution
- integration points that expose application systems as typed tools
- skills that package operating knowledge in a form the harness can use
- workspace protocols that make remote filesystem state durable
- logs, traces, evals, and contract tests that make behavior inspectable across
  runs and implementations

An agent system has three layers.

- The **definition layer** describes the agent: prompt, tools, policies, state,
  resources, and outputs.
- The **integration layer** connects definitions to a concrete harness, sandbox,
  workspace, tools, skills, and event stream.
- The **model-harness runtime** runs the model loop and built-in harness tools.

Application teams own the definition and integration layers: definitions,
harness adapters, tool bridges, skills, tests, evals, contract tests, and
operational evidence. Harness and sandbox platforms own the runtime. The line
between them must preserve work identity, bridge tools without broad network
access, expose a durable filesystem, stream observable events, and reconnect
without duplicating work.

## Design Map

These files form one contract.

- [PRINCIPLES.md](PRINCIPLES.md) states the constraints that govern the
  repository.
- [DEFINITION.md](DEFINITION.md) defines the portable definition model.
- [CAPABILITIES.md](CAPABILITIES.md) covers tools, policies, feedback,
  completion checking, and external access.
- [SKILLS.md](SKILLS.md) treats skills as versioned integration assets.
- [STATE.md](STATE.md) covers session state, resources, typed contracts, and
  transaction limits.
- [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md) describes the production topology:
  orchestrator, sandbox control plane, harness runtime, and protocol edge.
- [WORKSPACES.md](WORKSPACES.md) defines the remote filesystem model.
- [DURABLE-WORK.md](DURABLE-WORK.md) defines work identity, idempotency,
  reconnect, conflicts, and lifecycle separation.
- [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) covers harness adapters and
  integration points.
- [OBSERVABILITY.md](OBSERVABILITY.md) defines events, transcripts, snapshots,
  traces, and debug bundles.
- [EVALUATION.md](EVALUATION.md) covers agent loops, eval loops, datasets,
  prompt overrides, and contract tests.
- [GLOSSARY.md](GLOSSARY.md) names the core concepts.

## Scope

This repository sits between application code and any one model-harness runtime.
It asks:

- What must application teams own when the model and harness are built together?
- What belongs in an agent definition, and what belongs in the integration
  layer?
- Which integration points and skills create durable leverage?
- How does a client name work so retries and reconnects are safe?
- What does a remote persistent workspace need to provide?
- Where can transaction guarantees honestly hold?
- How are tool calls, built-in harness events, and filesystem changes observed?
- What must an integration layer prove before it can claim the shared contract?

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
sandbox correctness, remote correctness wins. Local mode uses the same protocol
rules in a simpler setting; it is not a separate architecture.
