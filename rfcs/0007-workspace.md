# RFC-0007: Workspace, Sandbox, and Egress

- Status: Draft
- Ring: environment
- Depends on: RFC-0001, RFC-0003

## Summary

The environment an agent acts on is declared as **data** on the definition —
mounts, posture, setup, egress rules, credential *names* — and materialized
by a provider at run time. The materialized environment is one aggregate with
one root, one lifecycle owner, and two effect facets: a filesystem over a
narrow backend protocol, and command execution that takes argument vectors,
never shell strings. Egress is default-deny. Secret material never appears
anywhere the definition or the agent can read, and the agent cannot widen its
own permissions.

## Motivation

Tools written against "the local filesystem" embed silent assumptions — host
paths, symlinks, shell expansion, ambient credentials, open networks — and
every one breaks when the workspace becomes a container, a remote sandbox, or
a customer-controlled environment. The definition must program against a
contract about the environment, not against the host. Unattended, that
contract is also the security boundary, so it must be declared, reviewable,
and enforced outside the agent's reach.

## Intent as data

A definition declares its environment as an inert, serializable value:
mounts (with include/exclude patterns, size budgets, and an allow-list of
host roots sources must live under), read-only or writable posture, setup
commands (each an argument vector), an egress policy, environment variables
layered over a hygienic base, and credential names. Because intent is data,
it is diffable in review, hashable for versioning (RFC-0013), and recorded in
the run record (RFC-0011).

Materialization is fail-closed: any error — an escaping mount, a failed setup
command, a blown budget — tears down the partially built environment before
propagating. There is no "mostly materialized" environment to run in.

## One aggregate, one owner

The materialized environment exposes an identity, a root, the two effect
facets, the egress control plane, snapshot/restore, and an idempotent
`close()` — the only teardown. Whoever opens it closes it; facets never
self-close.

A handle on the environment is a **lease**: for locally provisioned
environments, release destroys the directory; for pooled or harness-attached
environments, release detaches. The lease is the seam that lets the execution
substrate move — host process, container, remote box — without the definition
noticing. The aggregate deliberately separates the *workspace* (root, files,
snapshots — cheap, per-run) from the *substrate* (where processes run —
expensive, poolable), and implementations SHOULD keep that seam visible even
while both are local.

## The filesystem facet

All ergonomics — reads with pagination, streaming, write modes, size caps,
path normalization and validation, read-only enforcement — live once, in a
single facade. Beneath it sits a deliberately small backend protocol:
metadata, listing, search, ranged reads, writes, deletion, snapshot, restore
— and nothing else. Consequences:

- **Backends are cheap.** A fake, an in-memory backend, and a remote backend
  implement the same small protocol, and one shared validation suite runs
  against all of them — "workspace behavior independent of backing storage"
  is a tested property, not a slogan.
- **Search runs where the data lives.** Listing and content search are
  backend primitives so a remote backend executes them server-side.
- **Paths are workspace-relative**, normalized before the backend sees them;
  escapes are rejected uniformly.
- **Snapshots are opaque references**, all-or-nothing on restore: the
  workspace returns to the exact observed state — files created since are
  removed, whatever ignore conventions say. A restore that "mostly" rolls
  back breaks the tool transaction contract (RFC-0003).

## The command facet

Command execution takes an argument vector and executes it directly — no
shell interpretation, no globbing, no expansion. Working directories are
workspace-relative and escape-checked; output is capped with explicit
truncation flags; timeouts are mandatory with sane defaults; launch failures
map to conventional exit codes rather than exceptions, so the model always
gets a result it can reason about. Shell strings are an injection surface,
and an unattended agent composing them from model output is a known incident
category.

## Egress and credentials

- **Default deny.** An empty egress policy allows nothing; rules are explicit
  allows (host pattern, ports, protocol, optional credential to inject).
- **Credentials by name.** Definitions carry names; material is bound at run
  time by the operator side, injected only at the enforcement point, never
  written into the workspace, environment, logs, or serialized state, and
  cleared on close.
- **Not a tool surface.** Reconfiguring egress or credentials is never
  reachable through the agent's tool context. An agent MUST NOT be able to
  widen its own reach — this sentence is most of the prompt-injection
  blast-radius story.
- **Attested when delegated.** When a harness-provided environment enforces
  its own posture, the adapter records the *effective* posture in the run
  record, so gaps between declared intent and enforced reality are visible.

## Rendering from reality

Anything the definition shows the model about the workspace — a listing, a
preview — MUST derive from the materialized environment at render time, not
from copy-time bookkeeping. Before materialization, previews render an
explicit placeholder rather than a guess.

## Anti-patterns

- **Host paths in definitions.** A portability bug and usually a security
  bug.
- **The `sh -c` escape hatch.** One shell-string convenience reintroduces
  everything the argv contract removed.
- **Secrets in intent.** A credential value in a config file is a leak with a
  commit history.
- **Partial rollback.** Restoring tracked files while leaving untracked
  debris makes retries unsafe in the way that is hardest to debug.

## Compatibility surface

An adapter certification suite MUST assert:

- The same declared intent materializes to equivalent observable workspaces
  on every harness.
- Filesystem semantics are identical over every backend via the shared
  validation suite.
- Snapshot → restore returns the workspace to the exact prior state.
- Egress default-deny holds: undeclared destinations are unreachable.
- Secret material appears nowhere the agent or the run record can read.
