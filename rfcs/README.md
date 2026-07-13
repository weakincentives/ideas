# RFCs: The Agent Control Plane

Design RFCs for a state-of-the-art library for **unattended agents** — the
agent-definition layer and the control plane around it. Fully implementation
agnostic: contracts, semantics, invariants, and failure modes, never classes
or wire formats. Any implementation, in any language, claims conformance by
satisfying the normative requirements.

The narrative in one line: **one artifact defines the agent, one stream
records the run, explicit bounds contain it, tests prove it ports, and
evaluation improves it.** The root [README](../README.md) states the thesis
and the principles; these RFCs turn them into testable commitments.

## The three rings

1. **The definition** — the portable, reviewable artifact specifying what the
   agent *is*: instruction graph, tools, policies, feedback and completion,
   and the event ledger.
2. **The control plane** — the contracts that make unattended delegation
   safe: environment, capabilities, the execution envelope, work
   distribution, the run record, adapter certification, versioned iteration,
   and evaluation.
3. **The harness** — the runtime that executes the definition. Out of scope
   except where the adapter contract constrains it.

## Index

| RFC | Title | Ring |
| --- | --- | --- |
| [0001](0001-scope-and-boundary.md) | Scope and the Boundary | framing |
| [0002](0002-instruction-graph.md) | The Instruction Graph | definition |
| [0003](0003-tools.md) | Tools: The Transactional Side-Effect Boundary | definition |
| [0004](0004-policies.md) | Policies: Declarative Invariants over Workflows | definition |
| [0005](0005-feedback-and-completion.md) | Feedback and Completion Gates | definition |
| [0006](0006-event-ledger.md) | The Event Ledger | definition |
| [0007](0007-workspace.md) | Workspace, Sandbox, and Egress | environment |
| [0008](0008-capabilities-and-time.md) | Capabilities, Resources, and Injected Time | environment |
| [0009](0009-execution-envelope.md) | The Execution Envelope | control plane |
| [0010](0010-work-distribution.md) | Work Distribution | control plane |
| [0011](0011-run-record.md) | The Run Record | control plane |
| [0012](0012-adapter-contract.md) | Adapters and the Compatibility Suite | control plane |
| [0013](0013-versioning-and-overrides.md) | Versioned Iteration: Overrides and Experiments | control plane |
| [0014](0014-evaluation.md) | Evaluation as the Control Loop | control plane |

Read 0001 first; it frames everything and hoists the shared motivation so the
others don't repeat it. The definition ring (0002–0006) builds in order: the
graph carries tools, tools are gated by policies, feedback and completion
observe the trajectory, and everything lands on the ledger. The
environment RFCs (0007–0008) fix what definitions may assume about the world.
The control-plane RFCs (0009–0014) can be read in any order; 0012 references
everything else.

## Conventions

- **Normative language.** MUST, MUST NOT, SHOULD, and MAY in the RFC 2119
  sense.
- **Compatibility surface.** Each RFC ends by listing the observable
  behaviors the certification suite (RFC-0012) asserts. A requirement that
  cannot be phrased as observable behavior is a design smell.
- **Altitude.** RFCs specify observable contracts: behavior, invariants,
  failure semantics. Implementation choices that do not change behavior
  observable through the compatibility surface are left unstated on purpose —
  silence is freedom, not a gap.
- **Status.** All RFCs are Draft. An RFC becomes Stable when an
  implementation conforms and its compatibility surface is covered by a
  runnable suite.

## Decision log

Design forks resolved so far, folded into the RFCs as normative text:

1. **Policies gate the full action surface** (0004, 0012). Native harness
   tools are policy-gated via pre-action hooks with the same
   check/observe/deny-with-reason semantics; harnesses without an
   interception point declare the capability absent.
2. **The ledger is the substrate; the transcript is a view** (0006, 0011,
   0012). One append-only, high-granularity stream per run records everything
   — conversation, guardrail decisions, state transitions, operational
   signals. The transcript — what the model saw and did — is a deterministic
   projection of it and the compatibility oracle; state slices and metrics
   are other views, and not every state transition appears in the transcript.
   Definition-plane appends are authoritative; harness mirroring is
   best-effort. (Supersedes the earlier resolution that made the transcript
   itself the substrate.)
3. **Failure semantics belong to the definition** (0010, 0012). Retriability
   and dead-letter classes are declared against the typed error taxonomy;
   adapter error translation makes them portable; deployment owns mechanics.
4. **Analysis agents report, never act** (0014). Deliverables are structured
   findings with evidence references and machine-actionable proposals, acted
   on through the normal gates.
5. **The adapter floor is Core plus transcript emission** (0012). Below the
   floor an integration is not an adapter and is not certifiable.

## Glossary

| Term | Meaning |
| --- | --- |
| Definition | The portable artifact specifying what the agent is (0001) |
| Harness | The runtime executing a definition |
| Adapter | The thin, certified bridge from a definition to one harness (0012) |
| Section | A node in the instruction graph bundling prose and capabilities (0002) |
| Tool | A typed, transactional side-effect boundary (0003) |
| Policy | A fail-closed invariant gating actions (0004) |
| Completion gate | A check blocking termination until success criteria hold (0005) |
| Ledger | The run's single append-only, high-granularity event stream; the substrate all views derive from (0006) |
| Transcript | The view of what the model saw and did, projected deterministically from the ledger; the compatibility oracle (0006) |
| Slice | A typed state view folded from the ledger by pure reducers (0006) |
| Workspace | The environment the agent acts on, declared as data (0007) |
| Envelope | The time, budget, and liveness bounds on one run (0009) |
| Run record | The self-contained, queryable artifact explaining one run (0011) |
| Experiment | An immutable named bundle of definition variants (0013) |
