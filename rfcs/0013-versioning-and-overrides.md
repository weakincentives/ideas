# RFC-0013: Versioned Iteration — Overrides and Experiments

- Status: Draft
- Ring: control plane
- Depends on: RFC-0002, RFC-0011, RFC-0014

## Summary

Prompt text is the most-edited, least-protected part of most agent systems.
This RFC gives iteration a safety model: every overridable unit of a
definition carries a **content hash**; an **override** binds new text to the
hash it was written against and silently retires when the underlying content
changes; overrides group into named **tags**, and tags plus feature flags
into immutable **experiments** — the unit of comparison and rollout. The
rendered definition's identity travels in every run record, so any run is
attributable to exactly the variant that produced it.

## Motivation

One team edits code — with types, review, and version control. Another edits
behavior — prompt wording, tool descriptions, examples — and in most stacks
does so with none of those protections. The resulting failure is
stale-override drift: a tuned paragraph written against last month's
instructions is still being spliced into this month's, and nobody knows which
runs got which combination. Iteration must stay fast without becoming
untracked.

## Descriptors and overrides

The library computes a descriptor for every overridable unit — section
bodies, tool descriptions, parameter descriptions, worked examples — carrying
its position in the instruction graph and a **content hash** of its source
form. Deterministic rendering (RFC-0002) is what makes the hash meaningful;
an aggregate descriptor identifies the whole definition version.

An override names a target, the replacement content, and the hash it was
written against. On load: hash matches, the override applies; hash differs,
the override is **stale** — filtered out, the source content used, the
retirement visible in the run record. Fail closed in the direction that
matters: when in doubt, run the reviewed source, never a possibly orphaned
patch. Stale overrides surface as explicit re-basing work, not silent
behavior. Whatever the hashing granularity, staleness detection MUST err
toward retirement — a too-eager retirement costs a re-base; a too-long
survival costs correctness.

**The overridable set is bounded.** Text and examples: yes. Names, types,
structure, tool availability, policies, gates: no — those alter the
behavioral contract and go through the definition itself. The boundary keeps
overrides in "how it's phrased" and out of "what it can do," which is what
makes lightweight override review acceptable. Overrides target usages, not
shared sources: behavior is always reviewed in the context that ships it.

## Tags and experiments

Overrides group into named **tags** — coherent phrasing variants stored as
version-controlled data alongside the definition, resolved per run. A missing
tag falls back to source, loudly.

An **experiment** is an immutable named value: an overrides tag plus typed
feature flags, with sentinel baseline/control experiments anchoring
comparisons. Requests carry an experiment; the run resolves its variant from
it, records it on the transcript and in the run record, and reports it in
results. Flags parameterize definition-level choices — read by the
definition, never silently by the control plane. The experiment is the *only*
sanctioned mechanism for running the same agent differently in parallel;
ad-hoc config forks are the anti-pattern it replaces.

Every run record carries the aggregate descriptor, the tag and resolved
overrides (including stale-filtered ones), and the experiment. This closes
the attribution loop both ways: given any behavior, the exact rendered
definition that produced it is reconstructible; given any proposed override,
the population of affected runs is queryable.

## Optimization loops

Hash-anchored overrides are also the safe substrate for automated prompt
optimization: an optimizer proposes payloads against current hashes,
evaluation (RFC-0014) scores them under experiments, and promotion is a data
change with full provenance. The staleness machinery that protects human
edits protects machine edits — an optimizer's output written against last
week's definition retires itself.

## Anti-patterns

- **Free-floating patches.** Overrides keyed only by path, applied regardless
  of what the content now says — the stale-drift generator.
- **Structural overrides.** "Just one small extra tool" through the override
  channel: a capability change without contract review.
- **Mutable experiments.** Editing an experiment after runs have recorded it
  corrupts every comparison referencing it.
- **Config-branch forks.** Behavior variants via deployment conditionals —
  invisible to run records, incomparable by evaluation.

## Compatibility surface

An adapter certification suite MUST assert:

- The same definition with the same tag renders identically on every harness.
- A stale override is excluded on every harness, with the exclusion visible
  in the run record.
- Experiment identity flows unchanged from request to transcript to results
  to run record.
