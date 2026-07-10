# RFC-0011: The Run Record

- Status: Draft
- Ring: control plane
- Depends on: RFC-0006

## Summary

Every run must explain itself after the fact, to someone who wasn't there,
without the process that produced it. The **run record** is that explanation:
a self-contained, integrity-checked archive of one run — the ledger
(RFC-0006), state snapshots, configuration, environment, metrics, and
workspace — designed to be **queried** with standard tooling, because its
most important reader is a program, increasingly another agent.

## Motivation

Unattended operation moves debugging from "watch it happen" to "reconstruct
what happened." Harnesses each have native logs in native formats with native
gaps; a fleet running one definition across three harnesses would otherwise
need three forensic skill sets and produce three incomparable records. And
run volume means human reading is the exception: the record's design target
is programmatic analysis, with humans dropping in at the anomalies.

## Contents

A record contains, at minimum: the request and response; the full ledger,
from which the transcript view derives; state snapshots before and after; the definition's identity and version — tag,
resolved overrides, experiment (RFC-0013); the adapter and its declared
capabilities (RFC-0012); the declared and effective environment posture
(RFC-0007); the envelope and its consumption (RFC-0009); the error, if any,
with phase attribution; and, within declared size budgets, the workspace
contents.

For reproducibility, the record SHOULD capture the execution environment:
platform, runtime versions, dependency manifest, redacted environment
variables, and source-control position including uncommitted changes. "It
only fails in production" is answerable only if production described itself.

## Properties

- **Self-contained.** Diagnosis needs the record and standard tooling — not
  the original process, host, or dashboards.
- **Atomic.** The record exists completely or not at all; a crash during
  capture must not leave plausible-looking partial records.
- **Integrity-checked.** A manifest lists contents with checksums and a
  format version.
- **Gracefully degraded.** Capture failures (an oversized file, an unreadable
  path) are recorded as omissions, never turned into run failures.
- **Redacted by construction.** Secret material never enters the record
  (RFC-0007); environment capture is filtered by default.
- **Lifecycle-managed.** Retention policies and export to external storage
  are part of the contract; a fleet produces records faster than any disk
  grows.

## Correlation

Three identities appear in every record and every entry: the **request id**
(stable across redeliveries of one work item), the **run id** (fresh per
execution attempt), and the **attempt number** — plus pass-through slots for
distributed-tracing ids. The request/run distinction is what makes
at-least-once delivery (RFC-0010) auditable: one request, N runs, each with
its own record.

## Queryability

A record MUST answer structured questions with a standard query language and
no custom parsers: schema discovery, then queries over ledger events, the
transcript view, tool calls, errors, state slices (each typed slice as its
own relation), configuration, and metrics. Typed events make this cheap — the schema falls
out of the types. The query surface is the natural interface for **analysis
agents** (RFC-0014): an agent with a run record and a query tool can
investigate failures at fleet scale, which makes stable schemas and
discoverability hard requirements, not polish.

## Anti-patterns

- **Logs as the record.** Interleaved log lines without envelope, ordering,
  and correlation are data, not explanation.
- **Blocking telemetry.** Any path where capture failure fails the run
  inverts priorities.
- **Harness-native lock-in.** Treating one harness's format as the system of
  record makes every other harness second-class and cross-harness comparison
  impossible.
- **Unbounded capture.** Recording everything with no budgets or retention
  turns the forensic system into the outage.

## Compatibility surface

An adapter certification suite MUST assert:

- Records verify against their manifests on every harness.
- The ledger archived in a record equals the ledger emitted during the run,
  and the transcript view regenerates from it identically.
- Definition identity, adapter capabilities, envelope consumption, and
  correlation identities are present and correct.
- Induced capture failures degrade the record without failing the run.
