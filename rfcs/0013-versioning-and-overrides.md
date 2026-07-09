# RFC-0013: Versioned Iteration — Descriptors, Overrides, and Experiments

- Status: Draft
- Ring: control plane
- Depends on: RFC-0002, RFC-0011, RFC-0014

## Summary

Prompt text is the most-edited, least-protected part of most agent systems.
This RFC gives iteration a safety model: every overridable unit of a
definition has a **descriptor with a content hash**; an **override** binds new
text to a specific hash and is silently retired when the underlying content
changes; overrides group into named **tags**, and tags plus feature flags
group into immutable **experiments** — the unit of A/B comparison (RFC-0014)
and controlled rollout. The rendered definition's identity (its hashes, tag,
and experiment) travels in every run record, so any run is attributable to
exactly the definition variant that produced it.

## Motivation

Two teams edit an agent. One edits code — with types, review, tests, and
version control. The other edits behavior — prompt wording, tool
descriptions, examples — and in most stacks does so with none of those
protections, because the fast path (edit the string in production config)
bypasses them. The resulting failure is **stale-override drift**: a tuned
paragraph written against last month's instructions is still being spliced
into this month's, and nobody knows which runs got which combination.

The design goal is to make behavior iteration *fast without becoming
untracked*: prompt changes shouldn't require code deploys, but every change
must be anchored to what it modifies, reviewable, and attributable in run
records.

## Design

### Descriptors and hashes

The library computes a **descriptor** for every overridable unit of a
definition — section bodies, tool descriptions, tool parameter descriptions,
worked examples — carrying a stable path (the unit's position in the
instruction graph, RFC-0002) and a **content hash** of the unit's source
form. Deterministic rendering (RFC-0002) is what makes the hash meaningful.
An aggregate descriptor identifies the whole definition version.

### Overrides bound to hashes

An override names a target path, the replacement content, and **the hash of
the content it was written against**. On load:

- hash matches → the override applies;
- hash differs → the override is **stale**: it is filtered out, the source
  content is used, and the retirement is logged and visible in the run
  record.

Fail-closed, in the direction that matters: when in doubt, run the
*reviewed source*, never a possibly-orphaned patch. Stale overrides surface
in tooling as explicit "needs re-basing" work, not as silent behavior.

**The overridable set is bounded.** Text and examples: yes. Names, parameter
types, section structure, tool availability, policies, gates: no — those
changes alter the behavioral contract and MUST go through the definition
itself (code review, compatibility surface). The boundary keeps overrides in
"how it's phrased" and out of "what it can do", which is precisely what makes
lightweight override review acceptable.

### Tags

Overrides group into named **tags** — coherent variants of a definition's
phrasing ("assertive-feedback", "terse-v2"). Tags are stored as
version-controlled data alongside the definition (reviewable, diffable), keyed
by definition identity. Resolution is per-run: same definition, different tag,
different rendered text — same capability set. A missing tag falls back to
source, and SHOULD do so loudly enough to catch typos in deployment config.

### Experiments

An **experiment** is an immutable named value: an overrides tag plus a bag of
typed feature flags, with optional owner and description. Sentinel
baseline/control experiments (no overrides, no flags) anchor comparisons.

- Requests carry an experiment; the run resolves its definition variant from
  it, records it in the ledger and run record, and reports it in results.
- Flags parameterize definition-level choices (verbosity, alternate section
  enablement, model hints) — read by the definition, never silently by the
  control plane.
- Experiments compose with evaluation (RFC-0014): submit one dataset under N
  experiments, compare pass rates and scores per experiment, promote a
  winner by making its tag the default.

The experiment is deliberately the *only* sanctioned mechanism for "run the
same agent slightly differently in parallel" — ad-hoc environment-variable
forks are the anti-pattern it replaces.

### Identity in run records

Every run record (RFC-0011) MUST carry: the definition's aggregate
descriptor, the tag and resolved overrides (including which were filtered as
stale), and the experiment. This closes the attribution loop: given any
production behavior, the exact rendered definition that produced it is
reconstructible; given any proposed override, the population of runs it would
affect is queryable.

### Optimization loops

Hash-anchored overrides are also the safe substrate for *automated* prompt
optimization: an optimizer proposes override payloads against current hashes,
evaluation (RFC-0014) scores them under experiments, and promotion is a data
change with full provenance. The same staleness machinery that protects human
edits protects machine edits — an optimizer's output written against last
week's definition retires itself.

## Anti-patterns

- **Free-floating patches.** Overrides keyed only by path, applied regardless
  of what the content now says — the stale-drift generator.
- **Structural overrides.** "Just one small extra tool" through the override
  channel; capability changes without contract review.
- **Mutable experiments.** Renaming or editing an experiment after runs have
  recorded it corrupts every comparison that references it.
- **Config-branch forks.** Behavior variants via deployment config
  conditionals — invisible to run records, uncomparable by evaluation.

## Compatibility surface

An adapter certification suite MUST assert:

- The same definition with the same tag renders identically on every harness
  (extends RFC-0002's parity to overridden forms).
- A stale override (hash mismatch) is excluded on every harness, with the
  exclusion observable in the run record.
- Experiment identity appears in ledger, results, and run record.
