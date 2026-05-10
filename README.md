# The Many Habits of Highly Effective Agents

Guiding principles for agent systems that rent coupled model-harness execution
substrates and own the integration layer around them.

It is not an implementation, API reference, product roadmap, or set of docs
for any single existing project. The ideas are meant to guide future libraries
in any language. A good implementation should work in local development, but
the architecture should assume production agents run in remote sandboxes with
remote persistent workspaces.

## Thesis

Effective agents are not produced by a bare model call, a clever prompt, or a
single orchestration abstraction. They come from repeated engineering
disciplines: respecting the model-harness substrate, making capabilities
explicit, packaging operational knowledge as skills, keeping workspaces durable,
designing for interruption, and preserving evidence for every run.

The useful substrate is the model plus the harness it was trained,
post-trained, evaluated, and productized with: planning loop, native tools,
editing semantics, approval modes, transcript shape, filesystem expectations,
and recovery behavior.

That coupling is not a bug to abstract away. It is the capability being rented.
The wrong investment is rebuilding a generic planning loop or pretending every
harness is a thin wrapper around the same model API. The right investment is
the layer around the coupled substrate:

- **drivers** that connect application-owned work to harness-native execution
- **integration points** that expose application systems as typed tools
- **skills** that package operational knowledge for the harness to use well
- **remote workspace protocols** that make filesystem state durable and portable
- **observability and evals** that make harness behavior inspectable across
  runs and implementations

An unattended agent system therefore has three useful strata.

- The **definition layer** describes the agent: prompt structure, tools,
  policies, feedback, completion criteria, state, resources, and typed
  contracts.
- The **integration layer** binds the definition layer to a specific execution
  substrate, remote sandbox, persistent workspace, tool bridge, skill set, and
  event stream.
- The **execution substrate** runs the agent: model, harness loop, native tools,
  sandbox isolation, provider traffic, lifecycle, scheduling, retries, and
  recovery.

Application teams should own the definition, drivers, integrations, skills,
tests, and operational evidence. They should rent the execution substrate from
harness and sandbox platforms that evolve independently.

The central design requirement is that these strata meet through explicit
protocols. Those protocols must preserve durable work identity, expose a remote
filesystem, bridge tools without ambient network access, stream observable
events, and let clients reconnect without accidentally starting new work.

## Design Map

Read these files as one coherent contract.

- [PRINCIPLES.md](PRINCIPLES.md) lists the rules that govern the rest of the
  repository.
- [DEFINITION.md](DEFINITION.md) defines the portable agent definition model.
- [CAPABILITIES.md](CAPABILITIES.md) describes tools, policies, feedback,
  completion checking, and egress boundaries.
- [SKILLS.md](SKILLS.md) describes skills as versioned integration assets.
- [STATE.md](STATE.md) covers session state, resources, typed contracts, and
  transaction limits.
- [REMOTE-SANDBOXES.md](REMOTE-SANDBOXES.md) describes the production topology:
  orchestrator, sandbox control plane, harness runtime, and protocol boundary.
- [WORKSPACES.md](WORKSPACES.md) defines the remote persistent filesystem model.
- [DURABLE-WORK.md](DURABLE-WORK.md) defines work identity, idempotent turns,
  reconnect, conflict handling, and lifecycle separation.
- [INTEGRATION-LAYER.md](INTEGRATION-LAYER.md) explains how definition libraries
  bind to harness and sandbox protocols through drivers and integration points.
- [OBSERVABILITY.md](OBSERVABILITY.md) defines events, transcripts, snapshots,
  traces, and debug bundles.
- [EVALUATION.md](EVALUATION.md) covers agent loops, eval loops, datasets,
  prompt overrides, and conformance tests.
- [GLOSSARY.md](GLOSSARY.md) gives precise names for the concepts.

## Scope

This repository is concerned with the layer above a specific execution substrate
and below application code. It answers questions like:

- What should application teams own when the model and harness are coupled?
- What belongs in an agent definition, and what belongs in a driver?
- Which integration points and skills are worth building?
- How should a client name durable work so retries and reconnects are safe?
- What does a remote persistent workspace need to provide?
- Where can transaction guarantees honestly hold?
- How should tool calls, native tool events, and filesystem changes be
  observed?
- What should every driver prove before it conforms to the shared contract?

## Non-Goals

- It does not define a wire protocol in full detail.
- It does not prescribe a programming language.
- It does not assume one model provider, one harness, or one sandbox platform.
- It does not try to define a harness-independent planning loop.
- It does not require every implementation to expose every feature on day one.
- It does not claim that local execution is unimportant. Local execution is a
  useful special case, not the architecture to optimize around.

## Working Rule

When a future implementation has to choose between local convenience and remote
sandbox correctness, remote correctness wins. Local mode should be a degenerate
case of the same protocol semantics, not a separate architecture.
