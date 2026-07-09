# RFC-0002: The Instruction Graph

- Status: Draft
- Ring: definition
- Depends on: RFC-0001

## Summary

The agent's instructions are not a string template. They are a **typed,
hierarchical graph of sections**, where each section bundles the prose that
explains a capability with the capability itself — tools, knowledge packages,
local policies, and completion signals. Rendering the graph deterministically
produces both the text the model sees and the exact capability set the agent
has. Nothing the agent can do exists outside the rendered graph; nothing in
the rendered graph is undocumented.

## Motivation

Most agent stacks scatter the agent across artifacts that must be kept in sync
by convention: prompt text in one file, a tool registry assembled elsewhere,
availability rules in runtime glue, output schemas in a validator. Every layer
is separately maintained; they drift; and when behavior surprises someone, the
investigation is a scavenger hunt.

The failure is structural, not procedural. If documentation and capability are
different objects, they can disagree. The fix is to make them one object.

## Design

### Sections

A **section** is the unit of composition. Each section carries:

- **Instructions** — prose (or a template with typed placeholders) explaining
  what this part of the agent's job is and how to use its capabilities.
- **Children** — nested sections, forming a tree.
- **Attached capabilities** — tools (RFC-0003) and knowledge packages whose
  availability is tied to this section's presence in the rendered output.
- **An enablement predicate** — a function of current state deciding whether
  the section (and its entire subtree) exists in this rendering.
- **A visibility level** — fully rendered, or summarized for progressive
  disclosure (below).
- **Typed parameters** — placeholder values supplied as structured data,
  validated at construction time, not at model-call time.

Three consequences are normative:

1. **Co-location.** The section that documents a capability MUST be the
   section that attaches it. There is no separate tool registry to
   synchronize; documentation cannot drift from availability.
2. **Subtree scoping.** Disabling a section MUST remove its entire subtree —
   instructions, tools, and knowledge — from both the rendered text and the
   capability set, atomically. Instructions and capability change together or
   not at all.
3. **Structure over logic.** The templating surface SHOULD be minimal:
   placeholder substitution and boolean enablement. Conditional complexity
   belongs in the code that *builds* the graph (which is testable), not in a
   template language (which is not).

### Deterministic rendering

Rendering walks the graph and produces:

- **The instruction text** — with deterministic, hierarchical, numbered
  headings that mirror the graph structure, so the model's view corresponds
  to the author's structure and to positions that can be referenced in logs.
- **The capability set** — the ordered collection of tools and knowledge
  packages contributed by enabled sections.
- **The output contract** — the declared structured-output type, if any.

Rendering MUST be deterministic: the same definition, parameters, and state
produce byte-identical text and an identical capability set. Determinism is
what makes the rendered form hashable, and hashing is what anchors versioned
overrides (RFC-0013) and run-record attribution (RFC-0011).

Rendering failures — missing parameters, invalid placeholder types — MUST fail
loudly at render time. A definition that renders is a definition whose
declared inputs were satisfied.

### The rendered form is the review surface

The rendered definition — text plus capability set plus output contract — is
the **effective contract** for what the agent can see and do. Review tooling
SHOULD present it as a single document. An auditor asking "what can this agent
touch?" should read the rendered definition, not the runtime source.

### Progressive disclosure

Large agents accumulate reference material and advanced capabilities that are
only sometimes relevant. Shipping everything in every rendering wastes context
and diffuses attention; hiding things in external documents breaks the
co-location rule. Progressive disclosure resolves the tension:

- A section MAY render as a **summary** — a short description of what it
  contains — instead of its full body. Capabilities attached to a summarized
  section are **not** in the capability set.
- The library MUST provide a mechanism for the agent to request expansion of
  summarized sections, and a read-only variant for sections that carry no
  capabilities.
- Expansion is a first-class runtime event: it interrupts the current
  evaluation, applies a visibility override to run state, and re-renders. The
  expansion loop MUST be bounded to prevent pathological cycling.
- Expansion state lives in the state ledger (RFC-0006), so run records show
  exactly which parts of the definition the agent had revealed at each point.

This gives the definition author a dial: everything co-located and reviewable,
but context spent only when the agent decides a capability is relevant — and
that decision is itself recorded.

### Structured output as part of the definition

If the agent's deliverable is structured, the output type is declared **on the
definition**, not negotiated at call sites. The declaration covers the shape
(object or collection), the field types, and whether unknown fields are
tolerated. Parsing failures MUST surface the raw model output alongside the
validation error — an unattended pipeline cannot ask the model what it meant.

### Knowledge packages

Reference material that is bigger than instructions — procedures, examples,
document sets — SHOULD be attachable to sections as versioned packages with
validated metadata, following the same enablement and visibility rules as
tools. A knowledge package on a summarized section is invisible until the
section is expanded. Size limits and path hygiene are validation concerns of
the definition layer, so a definition that loads is a definition whose
knowledge attachments are well-formed.

## Failure modes addressed

- **Registry drift** — impossible by construction; there is no registry.
- **Capability without documentation** — impossible; attachment implies a
  documenting section.
- **Stale conditional logic** — enablement is data-driven and recorded, not
  buried in dispatch code.
- **Context bloat** — progressive disclosure spends context on demand.
- **Silent output-shape mismatch** — output contract is declared once and
  validated with loud failures.

## Anti-patterns

- **Template programming.** If a definition needs loops or conditionals in the
  template language, the graph should be restructured or built
  programmatically instead.
- **One mega-section.** A flat blob with all tools attached defeats scoping,
  disclosure, and review. Graph structure is the point.
- **Out-of-band instructions.** Any instruction delivered to the model outside
  the rendered definition (adapter-injected prose, harness-side prompt
  suffixes) undermines the review surface and MUST be limited to what the
  adapter contract explicitly allows (RFC-0012).

## Compatibility surface

An adapter certification suite MUST be able to assert:

- The same definition with the same enabled sections yields the same rendered
  instruction surface and the same capability schemas on every harness.
- Disabling a section removes its subtree's capabilities on every harness.
- Expansion requests round-trip: a summarized section's capability is absent,
  the expansion mechanism works, and the capability is present afterward.
- Declared structured output is parsed and validated identically, with parse
  failures carrying the raw output.

## Open questions

- Should sections support *mutually exclusive* groups (exactly one of N
  enabled) as a first-class construct, or is the enablement predicate enough?
- How much summary content is optimal before disclosure stops paying for
  itself? This is an evaluation question (RFC-0014) more than a design one.
- Should the rendered heading scheme be standardized across implementations to
  make cross-library run records comparable, or is per-implementation
  determinism sufficient?
