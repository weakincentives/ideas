# RFC-0012: Adapters and the Compatibility Suite

- Status: Draft
- Ring: control plane
- Depends on: all definition-ring RFCs, RFC-0007, RFC-0009, RFC-0011

## Summary

An adapter carries a definition to one harness. It is deliberately **thin**:
it translates transport, registration, streaming, and lifecycle — and it is
prohibited from reinterpreting anything the definition owns. Because every
adapter is a place where portability can quietly die, conformance is proven
by a **compatibility suite**: one parameterized set of behavioral scenarios
that every adapter runs, gated by an honest **capability declaration**, with
the canonical transcript (RFC-0011) as its oracle. Portability is a test
result, not a design intention.

## Motivation

The whole architecture rests on a claim: "the same definition behaves the
same on different harnesses." Nothing about writing an adapter makes that
claim true by default. Harnesses differ in tool protocols, hook semantics,
sandbox posture, streaming, and error taxonomy; a well-meaning adapter author
bridging those gaps makes dozens of small judgment calls, and each one is a
chance for "same definition" to decay into "similar behavior if nothing
unusual happens."

The failure pattern is known from every plugin ecosystem: per-adapter test
suites drift (a scenario added to one adapter never reaches the others),
capabilities get asserted in documentation rather than verified, and the gap
is discovered by the first production incident on the newer harness. The
remedy is also known: a single conformance suite, run by every
implementation, owned as the contract itself — the compatibility suite is to
this architecture what the specification test suite is to a language runtime.

## Design

### What an adapter is

One evaluation flows through a fixed sequence, and the adapter owns only the
translation steps: render the definition (once, via the library — RFC-0002);
map the rendered capability set onto the harness's tool/knowledge surface;
attach the definition's policy gates, feedback injection points, and
completion gates to the harness's hook or continuation mechanism — including
routing native harness actions through policy checks where the harness
exposes a pre-action interception point (RFC-0004); run the
harness against the materialized workspace (RFC-0007); translate the
harness's native stream into canonical events and transcript entries
(RFC-0006, RFC-0011); and parse structured output through the library.

The **runtime pairing** rule keeps lifecycles coherent: definition, adapter,
and materialized environment are bound together exactly once, and everything
in one run — including bounded internal retries such as progressive
disclosure re-renders (RFC-0002) and completion continuations (RFC-0005) —
executes against that same environment. Files written in round one are
visible in round two; the workspace lease spans the run and is released once
(RFC-0007). Making a mismatched pairing unrepresentable in the API is the
cheapest way to enforce this.

### What an adapter may not do

The prohibitions are the contract's teeth. An adapter MUST NOT:

- alter tool schemas, semantics, or availability beyond faithful translation;
- weaken, reorder, or skip policy checks, or execute a denied action;
- bypass completion gates or the envelope's checkpoints (RFC-0009);
- mutate state outside typed events, or emit events out of contract order;
- inject instructions into the model's context beyond declared mechanisms
  (feedback delivery, completion feedback — each of which is recorded);
- swallow harness errors into silence — every harness failure normalizes
  into the library's typed error taxonomy with phase attribution. This
  translation layer is load-bearing encapsulation, not logging hygiene:
  definitions declare retriability and dead-letter classifications against
  the typed taxonomy (RFC-0010), so a harness error that leaks through
  untranslated silently breaks failure handling.

Where a harness cannot support a contract at all, the adapter declares the
capability absent (below) rather than approximating it. **An honest gap is
compatible; a quiet approximation is not.**

### Capability declaration

Adapters declare what they support against a tiered vocabulary, roughly:

| Tier | Contents |
| --- | --- |
| Core | Evaluation, definition-tool bridging, structured output |
| Observability | Canonical events, transcript emission |
| Semantics | Transactions, progressive disclosure, envelope enforcement |
| Guardrails | Policy gating of definition tools, native-action gating, feedback delivery, completion gates |
| Environment | Sandbox posture, egress policy, workspace isolation, knowledge mounting |

Declarations MUST be conservative-by-default for the upper tiers so a new
adapter starts honest and expands as it earns each flag. The declaration is
machine-readable — it gates the suite, and it belongs in the run record
(RFC-0011) so an operator can see which guarantees a given run actually had.

**The floor.** An integration MUST support the Core tier plus transcript
emission to be called an adapter at all; below that, it is not certifiable
and none of the portability claims in this collection apply to it. The floor
is where it is because the transcript is both the substrate for state
(RFC-0006, RFC-0011) and the suite's oracle — an integration that cannot
emit it cannot be reasoned about, compared, or debugged.

### The compatibility suite

One suite, many adapters:

- **Scenarios are adapter-agnostic.** No scenario references a specific
  harness; each adapter contributes only a small fixture (construct adapter,
  report availability, declare capabilities). Scenario code importing an
  adapter's internals is a suite bug.
- **Capability gating, skip-don't-fail.** Scenarios requiring an undeclared
  capability skip visibly; an unavailable harness (no credentials, missing
  binary) skips entirely. Skips are visible in reporting — a sea of skips is
  itself information.
- **The transcript is the oracle.** Scenarios assert on canonical transcript
  structure and event sequences — the strongest adapter-agnostic evidence
  that the same thing happened. Suites MUST NOT assert on model prose
  quality; separating "the adapter works" from "the model did well" is what
  keeps the suite deterministic enough to run in CI. Model quality is
  evaluation's job (RFC-0014).
- **Determinism discipline.** Scenario prompts are engineered for minimal
  nondeterminism: constrained instructions, small output schemas, short
  expected behaviors. Where harness behavior still varies, scenarios assert
  invariants — ordering, envelope, rollback — not exact counts.
- **Certification is continuous.** The suite runs in CI per adapter, per
  change; a passing run at release time is the certificate. New scenarios
  reach every adapter automatically — that is the drift cure.

### Suite scope

The compatibility surfaces named by the other RFCs enumerate the assertions;
in outline the suite covers: rendered-surface parity; tool bridging and
schema parity; transactional rollback observed through both state and
workspace; policy denial delivery, including native-action gating where
declared; error-taxonomy normalization (the same induced failure classifies
identically across harnesses); feedback delivery and recording;
completion blocking, passing, and envelope bypass; deadline and budget
checkpoint behavior; transcript envelope/ordering/vocabulary; run-record
integrity; and environment posture (isolation, egress default-deny) where
declared.

## Anti-patterns

- **The helpful adapter.** Patching over a harness quirk by adjusting a tool
  result or reordering events "to be nice" — the drift begins as a favor.
- **Per-adapter scenario forks.** Copy-pasted suites with adapter-specific
  tweaks recreate exactly the divergence the suite exists to kill.
- **Aspirational capabilities.** Declaring a flag because support is planned;
  the declaration is a warranty, not a roadmap.
- **Quality assertions in conformance.** Suites that fail on model wording
  train everyone to ignore red builds.

## Compatibility surface

This RFC *is* the compatibility surface; its own meta-requirements:

- The suite MUST be runnable by third parties against a new adapter without
  modifying scenario code.
- Suite results MUST be reportable as a capability-by-scenario matrix.
- The library MUST version the suite; an adapter's certification names the
  suite version it passed.
