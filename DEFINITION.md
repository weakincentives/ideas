# Definition Model

An agent definition is a portable, typed description of what an agent is allowed
to think about and do. It should be stable across harnesses and sandbox
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
- native shell, file, search, and edit tools
- sandboxing and lifecycle
- provider credentials
- retry mechanics
- runtime-specific approval modes
- native event formats

The driver maps between them. It preserves the definition's application
semantics while respecting the execution substrate's native behavior.

## Prompt as Structure

A prompt should be rendered from typed structure, not assembled through ad hoc
string concatenation. The structure should support:

- stable ordering
- deterministic rendering
- nested sections
- conditional inclusion
- local tool declarations
- local resource requirements
- local policies and feedback
- structured output declarations
- content hashing for overrides and experiments

The rendered prompt is an artifact. It should be observable, hashable, and
attached to the work it produced.

## Sections

A section is the preferred unit of composition. It can carry instructions,
children, tools, policies, feedback providers, resources, examples, and
visibility metadata.

Sections make capability local. A tool should appear near the instructions that
teach the agent how to use it. A policy should appear near the behavior it
constrains. A completion check should appear near the output it validates.

## Progressive Disclosure

The definition should not flood the model with every possible detail up front.
It should support summary-first rendering and explicit expansion.

Progressive disclosure applies to text and capability:

- hidden sections may become visible after a typed expansion request
- detailed examples may be withheld until needed
- expensive tools may be unavailable until the agent enters the phase that
  needs them
- expansion events should be recorded

The goal is not secrecy. The goal is to keep the active context small while
preserving a complete, inspectable definition.

## Structured Output

When a run expects structured output, the definition should declare it as a
typed contract. The driver may use native schema enforcement when the harness
supports it. Otherwise it should fall back to prompt instructions plus
validation and repair.

The caller should receive a typed result, not unvalidated JSON-shaped text.

## Overrides

Prompt iteration should be possible without silently mutating source
definitions. An override should target a stable content hash or structural
path, carry authorship and experiment metadata, and fail loudly if the base
content changed.

Overrides are useful for evaluation and experimentation. They should not become
an unreviewed production configuration channel.

## Portability Test

A definition is portable when it can be rendered, inspected, evaluated, and run
against multiple drivers without changing definition code. Harness differences
belong in drivers, skill packages, and compatibility shims, not in prompt
branches.

Portability does not mean identical behavior across harnesses. It means the
same application intent can be driven through each execution substrate with the
differences made explicit, tested, and observable.
