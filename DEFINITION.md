# Agent Definitions

An agent definition is the middle layer. It says what the agent should do, what
it may use, which rules apply, what output is expected, and which data or
runtime requirements affect behavior.

The definition is not the application. It does not own product storage,
business logic, authorization systems, metric ownership, or final output
delivery.

The definition is not the harness. It does not own model calls, the model loop,
built-in tools, sandbox lifecycle, provider credentials, or runtime event
formats.

The definition can still declare requirements. For example, it may require a
persistent workspace, query access, structured output, approval before delivery,
or review when cost crosses a threshold. Declaring a requirement is different
from depending on one harness's private implementation.

## What It Declares

A definition declares:

- prompt structure
- sections and visibility rules
- declared tools
- required skills
- policies
- feedback
- completion checks
- structured output
- resource requirements
- data and query requirements
- state that is meaningful to the agent

Application-facing integration binds those declarations to real application
systems: tool handlers, authorization checks, data sources, metric definitions,
output consumers, and eval fixtures.

Runtime-facing integration renders those declarations into a real harness and
sandbox through a harness adapter.

## Prompt as Structure

A prompt is rendered from typed structure. It is not assembled through ad hoc
string concatenation.

The structure gives the renderer:

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

Sections keep related content together. A tool appears near the instructions that
teach the agent how to use it. A policy appears near the behavior it constrains.
A completion check appears near the output it validates.

## Progressive Disclosure

The definition does not have to flood the model with every detail up front. It
can render summaries first and expose details later.

Progressive disclosure applies to text and capability:

- hidden sections may become visible after an expansion request
- detailed examples may be withheld until needed
- expensive tools may be unavailable until the phase that needs them
- expansion events are recorded

This is not secrecy. It keeps active context small while preserving a complete
definition that can be inspected later.

## Data and Runtime Requirements

Analytical agents often depend on data and runtime conditions that are easy to
miss. A definition may need to declare:

- data sources or metrics the agent may use
- freshness rules
- privacy or row-level limits
- result-size expectations
- query cost limits
- required workspace shape
- required built-in harness tools
- required skill packages
- approval or review checks
- final delivery rules

These requirements make behavior explicit. The harness adapter and application
integration either satisfy them, deny the run, or report a visible limitation.

## Structured Output

When a run expects structured output, the definition declares the output type.
The harness adapter may use the harness's own schema support when it exists. If
not, it falls back to prompt instructions plus validation and repair.

The caller receives a typed result, not unvalidated JSON-shaped text.

## Overrides

Prompt iteration should not silently mutate source definitions. An override
targets a stable content hash or structural path, records authorship and
experiment metadata, and fails loudly if the base content changed.

Overrides are support machinery around the definition. They must not become an
unreviewed production configuration channel.

## Cross-Harness Readiness

A definition is ready for more than one harness when the same application
intent and declared capabilities can be rendered, inspected, evaluated, and run
against each supported harness adapter without changing definition code.

This does not mean identical behavior. It means the same intent can be driven
through each model-harness runtime, with differences made explicit, tested, and
visible.

It also does not mean flattening harnesses into the lowest common denominator.
Harness-specific rendering, skills, events, limits, and failure modes belong in
harness adapters, skill packages, and sandbox protocols, not in prompt branches.

A harness adapter should not fake unsupported behavior. If a definition requires
tool streaming, workspace snapshots, structured output, approval gates, or a
data-access path that a harness cannot support, that limit should be visible and
tested.
