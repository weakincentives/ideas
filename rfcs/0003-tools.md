# RFC-0003: Tools — The Transactional Side-Effect Boundary

- Status: Draft
- Ring: definition
- Depends on: RFC-0002, RFC-0006, RFC-0007

## Summary

Tools are the only place where an agent's reasoning touches the world. A
state-of-the-art library treats every tool call as a **typed, policy-gated,
transactional operation**: typed input, typed output, structured failure that
returns to the model instead of crashing the run, and atomic commit-or-rollback
semantics over both agent state and workspace. A failed tool leaves no trace.

## Motivation

In unattended operation, tool failures are not exceptional — they are the
normal texture of a long run. Three properties decide whether a run survives
them:

1. **Failures must be survivable.** An exception that aborts the evaluation
   turns a recoverable misstep into a dead run with a half-modified workspace.
2. **Failures must be clean.** If a failed tool leaves partial writes — a
   half-written file, a state entry recorded before the error — every retry
   compounds the damage and every post-hoc analysis is ambiguous.
3. **Failures must be legible.** The model has to be able to read what went
   wrong and reason about what to do differently; the operator has to be able
   to audit the same thing later.

Typed contracts and transactions are not ceremony; they are what makes
at-least-once execution (RFC-0010) and honest run records (RFC-0011) possible.

## Design

### The tool contract

A tool declares:

- **A name** — stable, machine-safe, unique within the rendered definition.
- **A description** — the model-facing explanation. Iterable via overrides
  (RFC-0013) without touching code; the name and types are not.
- **A typed parameter schema** and **a typed result schema** — structured
  types validated by the library, with unknown fields rejected by default.
  Loose maps and stringly-typed payloads are nonconforming.
- **Examples** — optional worked input/output pairs, validated against the
  schemas at definition-load time so examples cannot drift from the contract.
- **A handler** — the effectful function, which receives the parameters plus
  an injected **tool context** (below).

Tools attach to sections (RFC-0002); their availability is governed entirely
by the instruction graph.

### The tool result

Handlers return a structured result, never a bare value and never an
exception meant for the model:

- **Success or failure as data**, with a message the model reads.
- **A typed payload** on success; a payload MAY be withheld from the model's
  context (large or sensitive values) while still being recorded — with the
  explicit caveat that context exclusion is an ergonomics feature, not a
  security boundary.
- Failures are forwarded to the model as the tool's response. The run
  continues; the model decides what to do next.

A small set of signals is exempt from "never abort": envelope exhaustion
(deadline/budget, RFC-0009) and control-flow signals such as progressive
disclosure expansion (RFC-0002) propagate to the evaluation layer, because
they are decisions about the run, not about the tool.

### The tool context

Handlers MUST receive their dependencies through an injected context rather
than ambient globals. The context exposes, at minimum:

- the **workspace facets** — filesystem and command execution (RFC-0007),
- **resolved resources/capabilities** (RFC-0008),
- the **state ledger** for dispatching typed events (RFC-0006),
- the **envelope**: deadline, budget tracker, and a heartbeat the handler
  beats during long operations (RFC-0009),
- **correlation identifiers** for the run record (RFC-0011).

Handlers MUST NOT be able to observe which harness invoked them or where the
workspace physically lives. That opacity is the portability guarantee.

### Transactional execution

Every tool call follows one canonical sequence:

1. **Resolve** the tool by name from the rendered capability set.
2. **Parse and validate** arguments against the schema; reject unknown fields.
3. **Check policies** (RFC-0004); a denial returns a structured failure with
   the policy's explanation, without executing the handler.
4. **Check the envelope**; refuse if the deadline has passed or budget is
   exhausted.
5. **Snapshot** the transaction pair: agent state and workspace.
6. **Execute** the handler.
7. On success: **commit**, notify policies so they can update their state,
   and run feedback providers (RFC-0005).
8. On failure (structured failure or caught exception): **restore** the
   snapshot. The failed call remains on the run's transcript (the substrate
   is append-only) but MUST NOT leave effects in mutable state or the
   workspace.
9. **Record** the invocation — parameters, outcome, correlation id — in the
   event ledger.

The transaction boundary is the (state, workspace) pair. External systems a
tool touches (an HTTP API, a message broker) cannot be rolled back by the
library; tools with unrecoverable external effects SHOULD declare it, and
definitions SHOULD gate them with policies that make retries safe (e.g.,
idempotency keys recorded in state).

### Failure taxonomy

| Class | Handling |
| --- | --- |
| Invalid arguments | Structured failure to the model; no execution |
| Policy denial | Structured failure carrying the policy's reason |
| Handler error (expected or programming) | Structured failure; snapshot restored |
| Envelope exhaustion | Propagates; run-level handling (RFC-0009) |
| Control-flow signal (e.g., expansion) | Propagates; evaluation-layer handling |

The dispatch layer converting exceptions to structured failures MUST preserve
enough detail (type, message) for the run record while keeping the
model-facing message actionable.

### Idempotency

Delivery is at-least-once end to end (RFC-0010), so a whole run may re-execute
after a crash. Transactional tools guarantee a *clean* replay of state and
workspace, but not of external effects. The normative guidance: tools SHOULD
be idempotent where possible; where not, the definition owns making
re-execution safe — via policies, recorded evidence, or explicit
reconciliation steps — because only the definition knows the semantics.

## Anti-patterns

- **The god tool.** One `run_anything` tool with a free-text argument defeats
  typing, policy gating, and review. Tool surfaces should be small and sharp.
- **Exception-driven control flow.** Handlers raising exceptions to
  communicate normal outcomes (file missing, no matches) rob the model of
  legible feedback. Expected conditions are structured failures or successes.
- **Hidden effects.** A handler mutating state outside the ledger or writing
  outside the workspace facets breaks the transaction and the audit trail.
- **Harness sniffing.** Handlers branching on the detected runtime destroy
  portability precisely where it matters most.

## Compatibility surface

An adapter certification suite MUST assert, on every harness:

- The same tool schemas are exposed to the model for the same rendered
  definition.
- A tool returning failure leaves no workspace change and no mutable-state
  change (snapshot restore verified against both).
- Policy denials are delivered to the model as structured failures containing
  the reason, without handler execution.
- Every invocation emits a ledger event with a correlatable call identifier.
- Argument validation rejects unknown fields identically.
