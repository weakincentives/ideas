# RFC-0008: Capabilities, Resources, and Injected Time

- Status: Draft
- Ring: environment
- Depends on: RFC-0001, RFC-0003

## Summary

Everything a definition needs from the outside world — clients, stores,
trackers, clocks — is declared as a **typed capability** and resolved through
an explicit registry with scoped lifecycles. Nothing is ambient. Time itself
is a capability: monotonic time, wall-clock time, and sleeping are narrow
injectable protocols, and direct system-time calls are prohibited in the
definition layer. The payoff is the property that makes everything else in
this collection testable: **a definition's behavior is a function of its
declared inputs.**

## Motivation

Unattended agents are distributed systems, and the classic sins of
distributed-systems code — ambient singletons, hidden clients, direct clock
reads, sleeps sprinkled through logic — carry their usual costs plus one new
one: the definition stops being portable. A tool handler that reaches for a
global HTTP client with credentials from the host environment works on the
developer laptop and silently misbehaves in a remote sandbox.

There is also a testing imperative with unusual force here. Model calls are
expensive and nondeterministic, so the *rest* of the system must be cheap and
deterministic to test, or nothing ever gets tested. Deadlines, lease
extension, retry backoff, feedback cadence — all are time-driven behaviors
that are untestable against the real clock and trivially testable against an
injected one.

## Design

### Typed capabilities and the registry

- A **binding** associates a protocol (an interface type) with a provider
  function and a **scope**. The registry is immutable once built; only scope
  caches are mutable at run time.
- Resolution is **lazy** by default (construct on first use), with optional
  eager initialization for fail-fast startup.
- Providers may depend on other capabilities; the registry resolves the graph
  and MUST detect cycles with an error that names the path.
- Duplicate bindings are an error at composition time, not a
  last-writer-wins surprise.

**Scopes** are few and explicit:

| Scope | Lifetime | Typical use |
| --- | --- | --- |
| Run-scoped | One instance per run | Clients, configuration, budget tracker |
| Call-scoped | Fresh per tool call | Per-call tracers, request context |
| Transient | Fresh per access | Builders, buffers |

Lifecycle protocols round out the contract: post-construction initialization
(failures prevent caching and surface as provider errors), and disposal in
reverse construction order when the run's resource context closes.

### Layered contribution

Capabilities enter a definition at three layers, later layers overriding
earlier: the definition's own bindings, contributions from sections (a
section that documents a capability can also provide its client — co-location
again, RFC-0002), and per-run bindings supplied at invocation. This layering
is what makes the same definition runnable in production (real clients),
evaluation (recording clients), and unit tests (fakes) with zero definition
changes — swap the outermost layer only.

### Access

Tool handlers, feedback providers, and completion gates receive capabilities
through their injected context (RFC-0003) by asking for a protocol. Optional
lookups are explicit (`get-or-none`), so a definition's hard requirements are
distinguishable from its opportunistic ones — and the hard set is statically
enumerable for review: *what does this agent need to run?* is answerable from
the definition alone.

### Time as a capability

Time is decomposed into narrow protocols, and components MUST depend on the
narrowest one they need:

| Protocol | Provides | Used by |
| --- | --- | --- |
| Monotonic time | Elapsed-time measurement | Heartbeats, lease cadence, timeouts |
| Wall-clock time | Timezone-aware UTC now | Deadlines, event timestamps |
| Sleep (sync/async) | Delay | Pollers, backoff |

Two time domains exist because they answer different questions and fail
differently: monotonic time never goes backwards but means nothing across
processes; wall-clock time is comparable across systems but can jump.
Deadlines are wall-clock; intervals are monotonic; conflating them produces
bugs that appear only under NTP adjustment — i.e., only in production.

**Prohibition:** direct system-time reads and raw sleeps MUST NOT appear in
the definition layer or the control plane outside the system-clock
implementation itself. This is a lintable rule and SHOULD be enforced
mechanically.

The test double is a **fake clock** that advances instantly on sleep and
supports explicit advancement — turning "wait 10 minutes to observe lease
expiry" into a microsecond assertion. Every time-driven contract in this
collection (RFC-0005 cadences, RFC-0009 envelopes and leases, RFC-0010
visibility timeouts) is expected to be tested this way.

## Anti-patterns

- **The service locator.** A god object passed everywhere with getters for
  everything is ambient state with extra steps; contexts should expose the
  narrow set the component declared.
- **Config-driven conditionals.** Branching on environment names ("if prod")
  instead of binding different capabilities per environment reintroduces
  untestable divergence.
- **Clock mixing.** Comparing a monotonic reading with a wall-clock deadline;
  measuring elapsed time by subtracting wall-clock stamps.
- **Test-only seams.** If tests need a different wiring path than production
  (monkeypatching, private setters), the capability model has failed; the
  wiring path must be the same, with different bindings.

## Compatibility surface

Most of this RFC binds the library rather than adapters, but a certification
suite MUST assert:

- Tool contexts resolve the same declared capabilities on every harness.
- Capability lifecycle holds across a run: run-scoped instances are stable
  within a run, disposed at its end, on every harness.
- No harness leaks ambient time: injected fake clocks drive deadline and
  cadence behavior identically everywhere (verifiable with short synthetic
  deadlines).

## Open questions

- Qualifiers/named bindings (two instances of one protocol) versus wrapper
  types: wrapper types keep the model simple but multiply nominal types; is
  the simplicity worth it at fleet scale?
- Should capability *requirements* be declared statically on the definition
  (manifest-style) in addition to being discoverable from resolution, so
  review and deployment tooling can check satisfiability before a run starts?
- Async providers and structured-concurrency scopes: how much of the registry
  contract must change for fully async implementations?
