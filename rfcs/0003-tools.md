# RFC-0003: Tools — The Transactional Side-Effect Boundary

- Status: Draft
- Ring: definition
- Depends on: RFC-0002, RFC-0006, RFC-0007

## Summary

Tools are the only place where an agent's reasoning touches the world. Every
tool call is a **typed, policy-gated, transactional operation**: typed input,
typed output, structured failure that returns to the model instead of
crashing the run, and atomic commit-or-rollback over both agent state and
workspace. A failed tool leaves no trace.

## Motivation

In a long unattended run, tool failures are the normal texture, not the
exception. Three properties decide whether the run survives them: failures
must be **survivable** (no aborted evaluations over recoverable missteps),
**clean** (no partial writes to compound on retry), and **legible** (the
model reads what went wrong; the operator audits it later). Typed contracts
and transactions are not ceremony — they are what makes at-least-once
execution (RFC-0010) and honest run records (RFC-0011) possible.

## The tool contract

A tool declares a stable, machine-safe **name**; a model-facing
**description** (iterable via overrides, RFC-0013 — the name and types are
not); a **typed parameter schema** and **typed result schema** with unknown
fields rejected by default; optional **examples** validated against the
schemas at load time so they cannot drift; and a **handler**. Tools attach to
sections (RFC-0002); the instruction graph alone governs availability.

Handlers return a structured result, never a bare value or an exception meant
for the model: success or failure as data, a message the model reads, and a
typed payload. A payload MAY be withheld from model context while still being
recorded — an ergonomics feature, not a security boundary. Failures are
forwarded to the model as the tool's response; the run continues.

Two signal classes are exempt from "never abort": envelope exhaustion
(RFC-0009) and control-flow signals such as disclosure expansion (RFC-0002)
propagate to the evaluation layer, because they are decisions about the run,
not the tool.

## The tool context

Handlers receive their dependencies through an injected context, never
ambient globals: the workspace facets (RFC-0007), resolved capabilities
(RFC-0008), the ledger for dispatching typed events (RFC-0006), the
envelope — deadline, budget tracker, and a heartbeat to beat during long
operations (RFC-0009) — and correlation identifiers (RFC-0011).

Handlers MUST NOT be able to observe which harness invoked them or where the
workspace physically lives. That opacity is the portability guarantee.

## Transactional execution

Every call follows one canonical sequence:

1. **Resolve** the tool from the rendered capability set.
2. **Validate** arguments against the schema; reject unknown fields.
3. **Check policies** (RFC-0004); a denial returns a structured failure
   carrying the policy's reason, without executing the handler.
4. **Check the envelope**; refuse if deadline or budget is exhausted.
5. **Snapshot** the transaction pair: agent state and workspace.
6. **Execute** the handler.
7. On success: **commit**, notify policies, run feedback providers
   (RFC-0005). On failure: **restore** the snapshot — the attempt remains on
   the ledger, but no effect survives in state or workspace.
8. **Record** the invocation — parameters, outcome, correlation id — on the
   ledger.

The transaction boundary is the (state, workspace) pair. External systems a
tool touches cannot be rolled back by the library. Tools SHOULD be idempotent
where possible; where not, the definition owns making re-execution safe — via
policies, recorded evidence, or reconciliation steps — because only the
definition knows the semantics. This matters because delivery is
at-least-once end to end (RFC-0010): a whole run may replay after a crash,
and transactions guarantee a clean internal replay, not a clean external one.

## Failure taxonomy

| Class | Handling |
| --- | --- |
| Invalid arguments | Structured failure to the model; no execution |
| Policy denial | Structured failure carrying the policy's reason |
| Handler error | Structured failure; snapshot restored |
| Envelope exhaustion | Propagates; run-level handling (RFC-0009) |
| Control-flow signal | Propagates; evaluation-layer handling |

Exception-to-failure conversion MUST preserve enough detail for the run
record while keeping the model-facing message actionable.

## Anti-patterns

- **The god tool.** One `run_anything` with a free-text argument defeats
  typing, gating, and review. Tool surfaces are small and sharp.
- **Exception-driven control flow.** Expected conditions (file missing, no
  matches) are structured results, not exceptions — exceptions rob the model
  of legible feedback.
- **Hidden effects.** Mutating state outside the ledger or writing outside
  the workspace facets breaks the transaction and the audit trail.
- **Harness sniffing.** Handlers branching on the detected runtime destroy
  portability exactly where it matters most.

## Compatibility surface

An adapter certification suite MUST assert, on every harness:

- The same tool schemas are exposed for the same rendered definition.
- A failed tool leaves no workspace change and no working-state change
  (snapshot restore verified against both).
- Policy denials reach the model as structured failures with the reason,
  without handler execution.
- Every invocation lands on the ledger with a correlatable call id.
- Argument validation rejects unknown fields identically.
