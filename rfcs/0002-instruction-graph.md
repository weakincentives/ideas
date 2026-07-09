# RFC-0002: The Instruction Graph

- Status: Draft
- Ring: definition
- Depends on: RFC-0001

## Summary

The agent's instructions are not a string template. They are a typed,
hierarchical graph of **sections**, where each section bundles the prose that
explains a capability with the capability itself. Rendering the graph
deterministically produces both the text the model sees and the exact
capability set the agent has. Nothing the agent can do exists outside the
rendered graph; nothing in the rendered graph is undocumented.

## Motivation

Most agent stacks scatter the agent across artifacts kept in sync by
convention: prompt text here, a tool registry there, availability rules in
runtime glue, output schemas in a validator. They drift, and every behavioral
surprise becomes a scavenger hunt. The failure is structural: if
documentation and capability are different objects, they can disagree. The
fix is to make them one object.

## Sections

Each section carries: instructions (prose or a template with typed
placeholders, validated at construction time), child sections, attached
capabilities (tools per RFC-0003, knowledge packages), an enablement
predicate over current state, and a visibility level (full or summarized).

Three consequences are normative:

1. **Co-location.** The section that documents a capability MUST be the
   section that attaches it. There is no registry to synchronize, so
   documentation cannot drift from availability.
2. **Subtree scoping.** Disabling a section MUST remove its entire subtree —
   instructions, tools, knowledge — from the rendered text and the capability
   set atomically. What the agent reads and what it can do change together.
3. **Structure over logic.** Templating stays minimal: substitution and
   boolean enablement. Conditional complexity belongs in the code that builds
   the graph, which is testable — not in a template language, which is not.

## Deterministic rendering

Rendering walks the graph and produces the instruction text (hierarchical,
deterministic headings that mirror the structure), the ordered capability set
from enabled sections, and the declared output contract. The same definition,
parameters, and state MUST produce byte-identical text and an identical
capability set. Determinism makes the rendered form hashable; hashes anchor
versioned overrides (RFC-0013) and run-record attribution (RFC-0011).
Rendering failures — missing parameters, invalid types — fail loudly at
render time.

The rendered form is the review surface. Text plus capability set plus output
contract is the effective answer to "what can this agent see and do?" — an
auditor reads the rendered definition, not the runtime source.

## Progressive disclosure

Large agents accumulate reference material and advanced capabilities that are
only sometimes relevant. Shipping everything wastes context; hiding things in
external documents breaks co-location. Progressive disclosure resolves the
tension:

- A section MAY render as a **summary** of what it contains. Capabilities
  attached to a summarized section are not in the capability set.
- The agent can request expansion of summarized sections (with a read-only
  variant for sections carrying no capabilities). Expansion interrupts the
  current evaluation, applies a visibility override to run state, and
  re-renders; the loop MUST be bounded.
- Expansion state lives on the transcript (RFC-0006), so run records show
  exactly which parts of the definition the agent had revealed at each point.

The author gets a dial: everything co-located and reviewable, context spent
only when the agent decides a capability is relevant — and that decision is
itself recorded.

## Structured output

If the deliverable is structured, the output type is declared on the
definition, not negotiated at call sites: shape, field types, tolerance for
unknown fields. Parse failures MUST surface the raw model output alongside
the validation error — an unattended pipeline cannot ask the model what it
meant.

## Knowledge packages

Reference material bigger than instructions — procedures, examples, document
sets — SHOULD attach to sections as versioned packages with validated
metadata, following the same enablement and visibility rules as tools. A
definition that loads is a definition whose knowledge attachments are
well-formed.

## Anti-patterns

- **Template programming.** A definition that needs loops or conditionals in
  templates should restructure the graph or build it programmatically.
- **One mega-section.** A flat blob with every tool attached defeats scoping,
  disclosure, and review.
- **Out-of-band instructions.** Prose reaching the model outside the rendered
  definition undermines the review surface; it is limited to what the adapter
  contract explicitly allows (RFC-0012).

## Compatibility surface

An adapter certification suite MUST assert:

- The same definition with the same enabled sections yields the same rendered
  instruction surface and capability schemas on every harness.
- Disabling a section removes its subtree's capabilities on every harness.
- Expansion round-trips: a summarized section's capability is absent, the
  expansion mechanism works, and the capability is present afterward.
- Declared structured output is parsed and validated identically, with parse
  failures carrying the raw output.
