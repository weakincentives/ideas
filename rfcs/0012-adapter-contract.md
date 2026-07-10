# RFC-0012: Adapters and the Compatibility Suite

- Status: Draft
- Ring: control plane
- Depends on: RFC-0002 through RFC-0011

## Summary

An adapter carries a definition to one harness. It is deliberately thin — it
translates transport, registration, streaming, and lifecycle — and it is
prohibited from reinterpreting anything the definition owns. Because every
adapter is a place where portability can quietly die, conformance is proven
by a **compatibility suite**: one parameterized set of scenarios that every
adapter runs, gated by an honest **capability declaration**, with the
transcript (RFC-0006) as its oracle. Portability is a test result, not a
design intention.

## Motivation

The architecture rests on a claim — "the same definition behaves the same on
different harnesses" — that nothing about writing an adapter makes true by
default. Bridging harness differences involves dozens of judgment calls, and
each is a chance for "same definition" to decay into "similar behavior if
nothing unusual happens." Every plugin ecosystem shows the failure pattern:
per-adapter test suites drift, capabilities get asserted in documentation,
and the gap surfaces as a production incident. The remedy is also known: one
conformance suite, owned as the contract itself.

## What an adapter does

One evaluation flows through a fixed sequence; the adapter owns only the
translation steps. Render the definition once through the library (RFC-0002).
Map the capability set onto the harness's tool and knowledge surface. Attach
policy gates, feedback injection, and completion gates to the harness's hook
or continuation mechanism — including routing native harness actions through
policy checks where an interception point exists (RFC-0004). Run the harness
against the materialized workspace (RFC-0007). Translate the native stream
into canonical ledger events (RFC-0006). Normalize every harness failure
into the library's typed error taxonomy with phase attribution — load-bearing
encapsulation, since definitions declare failure semantics against that
taxonomy (RFC-0010). Parse structured output through the library.

**Runtime pairing:** definition, adapter, and materialized environment bind
together exactly once, and everything in one run — disclosure re-renders,
completion continuations — executes against that same environment. Files
written in round one are visible in round two; the workspace lease spans the
run and is released once. Making a mismatched pairing unrepresentable in the
API is the cheapest enforcement.

## What an adapter may not do

- Alter tool schemas, semantics, or availability beyond faithful translation.
- Weaken, reorder, or skip policy checks, or execute a denied action.
- Bypass completion gates or envelope checkpoints (RFC-0005, RFC-0009).
- Append definition-plane events out of contract, or mutate state outside
  typed events.
- Inject instructions beyond the declared mechanisms (feedback and completion
  delivery — each of which is recorded).
- Swallow harness errors into silence.

Where a harness cannot support a contract, the adapter declares the
capability absent rather than approximating it. **An honest gap is
compatible; a quiet approximation is not.**

## Capability declaration

Adapters declare support against a tiered vocabulary:

| Tier | Contents |
| --- | --- |
| Core | Evaluation, tool bridging, structured output |
| Observability | Canonical ledger events, transcript emission |
| Semantics | Transactions, disclosure, envelope enforcement |
| Guardrails | Policy gating (definition and native actions), feedback, completion gates |
| Environment | Sandbox posture, egress policy, isolation, knowledge mounting |

Declarations are conservative by default for the upper tiers, machine-
readable, and recorded in the run record so operators can see which
guarantees a run actually had. Capability names mean their *full* semantics —
native-action gating includes delivering the denial reason to the model; a
hook that can only deny silently does not earn the flag.

**The floor.** An integration MUST support Core plus transcript emission to
be called an adapter at all. Below the floor it is not certifiable and no
portability claim applies: the transcript view is the suite's oracle and the
minimum observability an operator needs — an integration that cannot produce
it cannot be compared or debugged.

## The suite

- **Scenarios are adapter-agnostic.** Each adapter contributes only a small
  fixture (construct, report availability, declare capabilities); a scenario
  importing adapter internals is a suite bug.
- **Capability gating, skip-don't-fail.** Undeclared capabilities skip
  visibly; an unavailable harness skips entirely. A sea of skips is itself
  information.
- **The transcript is the oracle.** Scenarios assert on the transcript view —
  what the model saw and did — the strongest adapter-agnostic evidence that
  the same thing happened; full-ledger granularity may legitimately differ
  across harnesses (RFC-0006). Suites MUST NOT assert on model prose quality;
  separating "the adapter works" from "the model did well" is what keeps
  conformance deterministic enough for CI. Model quality is evaluation's job
  (RFC-0014).
- **Determinism discipline.** Scenario prompts minimize nondeterminism;
  where harness behavior still varies, scenarios assert invariants —
  ordering, envelope, rollback — not exact counts.
- **Certification is continuous.** The suite runs in CI per adapter, per
  change; new scenarios reach every adapter automatically. That is the drift
  cure.

In outline, the suite covers: rendered-surface parity; tool schema parity;
transactional rollback observed through state and workspace; policy denial
delivery, including native-action gating where declared; error-taxonomy
normalization (the same induced failure classifies identically); feedback
delivery and recording; completion blocking, passing, and envelope bypass;
deadline and budget checkpoints; ledger envelope, ordering, and vocabulary;
transcript-view parity; run-record integrity; and environment posture where
declared.

## Anti-patterns

- **The helpful adapter.** Patching a harness quirk by adjusting a tool
  result or reordering events "to be nice" — drift begins as a favor.
- **Per-adapter scenario forks.** Copy-pasted suites recreate exactly the
  divergence the suite exists to kill.
- **Aspirational capabilities.** Declaring a flag because support is planned;
  the declaration is a warranty, not a roadmap.
- **Quality assertions in conformance.** Suites that fail on model wording
  train everyone to ignore red builds.

## Compatibility surface

This RFC is the compatibility surface; its meta-requirements:

- The suite MUST be runnable by third parties against a new adapter without
  modifying scenario code.
- Results MUST be reportable as a capability-by-scenario matrix.
- The suite is versioned; a certification names the suite version it passed.
