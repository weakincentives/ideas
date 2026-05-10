# Agent Definitions

An agent definition is a typed description of what the agent is supposed to do,
what it can use, which rules apply, and what output is expected. It can run
through more than one harness without pretending all harnesses are the same.

## What the Definition Owns

The definition owns:

- prompt structure
- sections and visibility rules
- declared tools
- policies
- feedback
- completion checks
- structured output
- state reducers
- resource requirements
- eval fixtures and prompt overrides

The harness owns:

- model calls
- planning and acting loop
- built-in shell, file, search, and edit tools
- sandboxing and lifecycle
- provider credentials
- retry behavior
- approval modes
- built-in event formats

The integration layer maps between the two. It preserves application intent
while respecting how the model-harness runtime works.

## Prompt as Structure

A prompt is rendered from typed structure, not assembled through ad hoc string
concatenation. The structure gives the renderer:

- stable ordering
- nested sections
- conditional inclusion
- local tool declarations
- local resource requirements
- local policies and feedback
- structured output declarations
- content hashes for overrides and experiments

The rendered prompt is an inspectable record attached to the work it produced.

## Sections

A section is the main unit of composition. It can carry instructions, child
sections, tools, policies, feedback, resources, examples, and visibility rules.

Sections keep related things together. A tool appears near the instructions that
teach the agent how to use it. A policy appears near the behavior it constrains.
A completion check appears near the output it validates.

## Progressive Disclosure

The definition does not have to flood the model with every detail up front. It
can render summaries first and expose details later.

Progressive disclosure applies to text and capability:

- hidden sections may become visible after an expansion request
- detailed examples may be withheld until needed
- expensive tools may be unavailable until the agent reaches the phase that
  needs them
- expansion events are recorded

This is not secrecy. It keeps active context small while preserving a complete
definition that can be inspected later.

## Structured Output

When a run expects structured output, the definition declares the output type.
The harness adapter may use the harness's own schema support when it exists. If
not, it falls back to prompt instructions plus validation and repair.

The caller receives a typed result, not unvalidated JSON-shaped text.

## Overrides

Prompt iteration does not silently mutate source definitions. An override
targets a stable content hash or structural path, records authorship and
experiment metadata, and fails loudly if the base content changed.

Overrides are for evaluation and experiments. They must not become an
unreviewed production configuration channel.

## Cross-Harness Test

A definition is ready when it can be rendered, inspected, evaluated, and run
against more than one harness adapter without changing definition code. Harness
differences belong in harness adapters, skill packages, and protocol shims, not
in prompt branches.

Cross-harness support does not mean identical behavior. It means the same
application intent can be driven through each model-harness runtime, with the
differences made explicit, tested, and visible.
