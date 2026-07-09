# RFCs: The Agent Control Plane

This directory holds a collection of design RFCs for a state-of-the-art library
for **unattended agents** — the agent-definition layer and the control plane
around it. The RFCs are deliberately **implementation agnostic**: they specify
contracts, semantics, invariants, and failure modes, not classes, languages, or
wire formats. Any implementation, in any language, should be able to claim
conformance with an RFC by satisfying its normative requirements.

The root [README](../README.md) states the thesis: the harness is the
depreciating asset; the definition is the durable one. These RFCs turn that
thesis into a set of specific, testable architectural commitments.

## The three rings

The RFCs organize the problem into three rings:

1. **The definition** — the portable, reviewable, typed artifact that specifies
   what the agent *is*: its instruction graph, tools, policies, feedback,
   completion criteria, and state contract.
2. **The control plane** — the contracts a library must provide so definitions
   can run unattended: the execution envelope (time, budget, liveness), work
   distribution, the run record, versioned iteration, evaluation, and adapter
   certification. The control plane is not the harness; it is the set of
   guarantees that make any harness safe to delegate to.
3. **The harness** — the runtime that executes the definition: planning loop,
   model calls, sandboxing, tool dispatch, scheduling, recovery. Out of scope
   for these RFCs except where a contract constrains it through the adapter.

## Index

| RFC | Title | Ring |
| --- | --- | --- |
| [0001](0001-scope-and-boundary.md) | Scope and the Definition/Harness Boundary | framing |
| [0002](0002-instruction-graph.md) | The Instruction Graph | definition |
| [0003](0003-tools.md) | Tools: The Transactional Side-Effect Boundary | definition |
| [0004](0004-policies.md) | Policies: Declarative Invariants over Workflows | definition |
| [0005](0005-feedback-and-completion.md) | Feedback and Completion Gates | definition |
| [0006](0006-state-ledger.md) | State as an Event Ledger | definition |
| [0007](0007-workspace.md) | Workspace, Sandbox, and Egress | environment |
| [0008](0008-capabilities-and-time.md) | Capabilities, Resources, and Injected Time | environment |
| [0009](0009-execution-envelope.md) | The Execution Envelope: Deadlines, Budgets, Leases, Liveness | control plane |
| [0010](0010-work-distribution.md) | Work Distribution: Queues, Dead Letters, and Shutdown | control plane |
| [0011](0011-observability.md) | The Run Record: Transcripts, Bundles, and Queryability | control plane |
| [0012](0012-adapter-contract.md) | Adapters and the Compatibility Suite | control plane |
| [0013](0013-versioning-and-overrides.md) | Versioned Iteration: Descriptors, Overrides, and Experiments | control plane |
| [0014](0014-evaluation.md) | Evaluation as the Control Loop | control plane |

## Reading order

RFC-0001 frames everything and should be read first. After that, the
definition-ring RFCs (0002–0006) build on each other in order: the instruction
graph carries tools; tools are gated by policies; feedback and completion
observe the trajectory; and all of it is recorded in the state ledger. The
environment RFCs (0007–0008) specify what definitions may assume about the
world they act on. The control-plane RFCs (0009–0014) can be read in any
order, though 0012 (compatibility) references almost everything else.

## Conventions

- **Normative language.** MUST, MUST NOT, SHOULD, and MAY are used in the
  RFC 2119 sense. A conforming implementation satisfies every MUST; a
  high-quality one satisfies the SHOULDs or documents why not.
- **Compatibility surface.** Most RFCs end with a "Compatibility surface"
  section listing the observable behaviors an adapter certification suite
  (RFC-0012) must assert. If a requirement cannot be phrased as an observable
  behavior, it is a design smell in the RFC.
- **Altitude.** RFCs specify observable contracts: behavior, invariants, and
  failure semantics. Implementation choices that do not change behavior
  observable through the compatibility surface are left unstated on purpose —
  silence is freedom, not a gap.
- **Status.** All RFCs are currently **Draft**. An RFC becomes **Stable**
  when at least one implementation conforms and the compatibility surface is
  covered by a runnable suite.

## Decision log

Design forks resolved so far, folded into the RFCs as normative text:

1. **Policies gate the full action surface** (RFC-0004, RFC-0012). Native
   harness tools are policy-gated via adapter pre-action hooks with the same
   check/observe/deny-with-reason semantics; harnesses without an
   interception point declare the capability absent rather than approximate.
2. **The transcript is the substrate** (RFC-0006, RFC-0011). One append-only
   event stream per run is the storage abstraction; the conversation view,
   the state view (reducer-folded slices), and the operational view are
   projections of it. Definition-plane appends are authoritative;
   harness-derived mirroring is best-effort.
3. **Failure semantics belong to the definition** (RFC-0010, RFC-0012).
   Retriability and dead-letter classification are declared by the definition
   against the library's typed error taxonomy; the adapter's error-translation
   layer provides the encapsulation that makes them portable; deployment owns
   the mechanics (queues, delivery counts, retention).
4. **Analysis agents report, never act** (RFC-0014). Their deliverable is a
   structured finding — conclusion, evidence references, machine-actionable
   proposal payloads — acted on through the normal gates.
5. **The adapter floor is Core tier plus transcript emission** (RFC-0012).
   Below the floor an integration is not an adapter and is not certifiable.

## Glossary

| Term | Meaning |
| --- | --- |
| Definition | The portable artifact specifying what the agent is (RFC-0001) |
| Harness | The runtime executing a definition (planning loop, sandboxing, dispatch) |
| Adapter | The thin, certified bridge from a definition to one harness (RFC-0012) |
| Section | A node in the instruction graph bundling instructions and capabilities (RFC-0002) |
| Tool | A typed, transactional side-effect boundary (RFC-0003) |
| Policy | A fail-closed invariant gating actions (RFC-0004) |
| Feedback provider | An observer injecting advisory guidance mid-run (RFC-0005) |
| Completion gate | A check that blocks termination until success criteria hold (RFC-0005) |
| Transcript | The run's single append-only event stream — every input, output, and action; the substrate all views derive from (RFC-0011) |
| Event ledger | The typed state view folded from the transcript by reducers (RFC-0006) |
| Workspace | The environment the agent acts on, declared as data (RFC-0007) |
| Envelope | The time/budget/liveness bounds on one run (RFC-0009) |
| Run record | The self-contained, queryable artifact explaining one run (RFC-0011) |
| Experiment | A named, immutable bundle of definition variants (RFC-0013) |
