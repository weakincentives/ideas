# RFC-0008: Capabilities, Resources, and Injected Time

- Status: Draft
- Ring: environment
- Depends on: RFC-0001, RFC-0003

## Summary

Everything a definition needs from the outside world — clients, stores,
trackers, clocks — is declared as a **typed capability** and resolved through
an explicit registry with scoped lifecycles. Nothing is ambient. Time itself
is a capability: monotonic time, wall-clock time, and sleeping are narrow
injectable protocols, and direct system-time calls are prohibited. The payoff
is the property everything else depends on: **a definition's behavior is a
function of its declared inputs.**

## Motivation

Unattended agents are distributed systems, and ambient singletons, hidden
clients, and raw clock reads carry the usual costs plus one more: the
definition stops being portable. There is also a testing imperative with
unusual force — model calls are expensive and nondeterministic, so the rest
of the system must be cheap and deterministic to test, or nothing gets
tested. Deadlines, lease cadence, backoff, and feedback timing are all
time-driven behaviors, untestable against the real clock and trivial against
an injected one.

## Capabilities

A **binding** associates a protocol (an interface type) with a provider and a
**scope**. Resolution is lazy by default with optional eager initialization;
providers may depend on other capabilities, and the registry resolves the
graph, detecting cycles with an error that names the path. Duplicate bindings
are a composition-time error, not last-writer-wins.

Scopes are few and explicit:

| Scope | Lifetime | Typical use |
| --- | --- | --- |
| Run | One instance per run | Clients, configuration, budget tracker |
| Call | Fresh per tool call | Per-call tracers, request context |
| Transient | Fresh per access | Builders, buffers |

Lifecycle completes the contract: optional post-construction initialization
(failures surface as provider errors and prevent caching) and disposal in
reverse construction order when the run closes.

Capabilities enter at three layers, later overriding earlier: the
definition's own bindings, contributions from sections (the section that
documents a capability can provide its client — co-location again,
RFC-0002), and per-run bindings at invocation. The layering is what lets one
definition run in production, evaluation, and unit tests with zero changes —
swap only the outermost layer.

Handlers, feedback providers, and gates resolve capabilities through their
injected context by protocol. Optional lookups are explicit, so the
definition's hard requirements are distinguishable from opportunistic ones —
and enumerable from the definition alone: "what does this agent need to run?"
has a static answer.

## Time

Time decomposes into narrow protocols, and components depend on the
narrowest they need:

| Protocol | Provides | Used by |
| --- | --- | --- |
| Monotonic time | Elapsed measurement | Heartbeats, lease cadence, timeouts |
| Wall clock | Timezone-aware UTC now | Deadlines, event timestamps |
| Sleep | Delay | Pollers, backoff |

Two domains exist because they fail differently: monotonic never goes
backwards but means nothing across processes; wall-clock is comparable across
systems but can jump. Deadlines are wall-clock; intervals are monotonic;
conflating them produces bugs that appear only under clock adjustment — that
is, only in production.

**Prohibition:** direct system-time reads and raw sleeps MUST NOT appear in
the definition layer or the control plane outside the system clock itself.
This is a lintable rule and SHOULD be enforced mechanically. The test double
is a fake clock that advances instantly on sleep — turning "wait ten minutes
to observe lease expiry" into a microsecond assertion. Every time-driven
contract in this collection is expected to be tested this way.

## Anti-patterns

- **The service locator.** A god object with getters for everything is
  ambient state with extra steps; contexts expose the declared narrow set.
- **Config-driven conditionals.** Branching on environment names ("if prod")
  instead of binding different capabilities reintroduces untestable
  divergence.
- **Clock mixing.** Comparing a monotonic reading to a wall-clock deadline,
  or measuring elapsed time from wall-clock stamps.
- **Test-only seams.** If tests need a different wiring path than production,
  the capability model has failed; same path, different bindings.

## Compatibility surface

An adapter certification suite MUST assert:

- Tool contexts resolve the same declared capabilities on every harness.
- Run-scoped instances are stable within a run and disposed at its end, on
  every harness.
- No harness leaks ambient time: injected clocks drive deadline and cadence
  behavior identically everywhere.
