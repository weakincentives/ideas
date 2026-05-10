# Definition Model

An agent definition is a portable, typed description of what an agent is allowed
to think about and do. It remains stable across harnesses and sandbox
platforms without pretending that all harnesses behave the same.

## Definition Boundary

The definition owns:

- prompt hierarchy
- sections and visibility rules
- declared tools
- policies
- feedback providers
- completion checks
- structured output contracts
- state reducers
- resource requirements
- evaluation fixtures and overrides

The harness owns:

- model invocation
- planning and act loop
- built-in shell, file, search, and edit tools
- sandboxing and lifecycle
- provider credentials
- retry mechanics
- runtime-specific approval modes
- built-in event formats

The integration layer maps between them. It preserves the definition's
application behavior while respecting how the model-harness runtime works.

## Prompt as Structure

A prompt is rendered from typed structure, not assembled through ad hoc string
concatenation. The structure supports:

- stable ordering
- deterministic rendering
- nested sections
- conditional inclusion
- local tool declarations
- local resource requirements
- local policies and feedback
- structured output declarations
- content hashing for overrides and experiments

The rendered prompt is an inspectable, hashable record attached to the work it
produced.

## Sections

A section is the preferred unit of composition. It can carry instructions,
children, tools, policies, feedback providers, resources, examples, and
visibility metadata.

Sections make capability local. A tool appears near the instructions that teach
the agent how to use it. A policy appears near the behavior it constrains. A
completion check appears near the output it validates.

## Progressive Disclosure

The definition does not flood the model with every possible detail up front. It
supports summary-first rendering and explicit expansion.

Progressive disclosure applies to text and capability:

- hidden sections may become visible after a typed expansion request
- detailed examples may be withheld until needed
- expensive tools may be unavailable until the agent enters the phase that
  needs them
- expansion events are recorded

This is not secrecy. It keeps the active context small while preserving a
complete, inspectable definition.

## Structured Output

When a run expects structured output, the definition declares it as a typed
contract. The harness adapter may use the harness's own schema support when it
exists. Otherwise it falls back to prompt instructions plus validation and
repair.

The caller receives a typed result, not unvalidated JSON-shaped text.

## Overrides

Prompt iteration happens without silently mutating source definitions. An
override targets a stable content hash or structural path, carries authorship
and experiment metadata, and fails loudly if the base content changed.

Overrides support evaluation and experimentation. They must not become an
unreviewed production configuration channel.

## Portability Test

A definition is portable when it can be rendered, inspected, evaluated, and run
against multiple harness adapters without changing definition code. Harness
differences belong in harness adapters, skill packages, and protocol shims, not
in prompt branches.

Portability does not mean identical behavior across harnesses. It means the
same application intent can be driven through each model-harness runtime with the
differences made explicit, tested, and observable.
