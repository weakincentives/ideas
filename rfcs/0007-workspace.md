# RFC-0007: Workspace, Sandbox, and Egress — Environment as Declared Intent

- Status: Draft
- Ring: environment
- Depends on: RFC-0001, RFC-0003

## Summary

The environment an agent acts on is declared as **data** on the definition —
mounts, posture, setup, egress rules, credential *names* — and materialized by
a provider at run time. The materialized environment is one aggregate with one
root, one lifecycle owner, and two effect facets: a **filesystem** built on a
narrow backend protocol, and **command execution** that takes argument vectors,
never shell strings. Network egress is default-deny. Secret material never
appears in the definition, the configuration, or the environment; the agent
cannot widen its own permissions. Snapshot and restore of the workspace are
part of the contract, because tool transactions (RFC-0003) depend on them.

## Motivation

Tools written against "the local filesystem" embed dozens of silent
assumptions: paths are host paths, symlinks resolve somewhere sensible, the
shell expands globs, credentials sit in the home directory, the network is
open. Every one of those assumptions breaks when the workspace becomes a
container, a remote sandbox, a virtual filesystem, or a customer-controlled
environment — and unattended agents are heading to exactly those places.

The definition must therefore program against a **contract about the
environment**, not against the host. And because the agent is unattended, the
environment contract is also the security boundary: what the agent can touch,
where its traffic can go, and what secrets it can exercise must be declared,
reviewable, and enforced outside the agent's own reach.

## Design

### Intent as data

A definition declares its environment needs as an inert, serializable value:

- **Mounts** — what gets copied or mapped in, with include/exclude patterns,
  size budgets, and an allow-list of host roots that sources must live under.
- **Posture** — read-only vs. writable filesystem.
- **Setup** — commands run in order after materialization, each an argument
  vector.
- **Egress policy** — allow rules (below); empty means nothing.
- **Environment variables** — layered over a hygienic base, never inherited
  wholesale from the host.
- **Credential names** — references only, never material.

Because intent is data, it is diffable in review, hashable for versioning
(RFC-0013), and recordable in the run record (RFC-0011).

**Fail-closed materialization**: any error while materializing — a mount
escaping its allowed root, a setup command failing, a budget exceeded — MUST
tear down the partially built environment before propagating. There is no
"mostly materialized" environment for an agent to run in.

### One aggregate, one owner

The materialized environment is a single aggregate exposing:

- an identity and a root,
- the two effect facets (filesystem, command execution),
- the egress control plane,
- snapshot/restore,
- `close()` — idempotent, the only teardown.

Whoever opens the environment closes it; facets never self-close. A handle on
the environment is a **lease**: for locally provisioned environments, release
destroys the directory; for pooled or harness-attached environments, release
detaches. The lease abstraction is the seam that lets the execution substrate
move (host process → container → remote box) without the definition noticing.

The aggregate deliberately separates two roles that local execution conflates:
the **workspace** (root, files, snapshots — cheap, per-run) and the
**execution substrate** (where processes actually run — expensive, poolable).
Implementations SHOULD keep the seam visible even while both are local.

### The filesystem facet: a narrow waist

All ergonomics — text/byte reads with pagination, streaming, write modes,
size caps, path normalization and validation, read-only enforcement — live
**once**, in a single facade. Beneath it sits a deliberately small backend
protocol: metadata, listing, search, ranged reads, writes, deletion,
snapshot, restore — and nothing else.

Consequences that are normative:

- **Backends are cheap.** A usable fake, an in-memory backend, and a remote
  backend all implement the same small protocol; a shared validation suite
  runs against every backend, so "workspace behavior independent of backing
  storage" is a tested property, not a slogan.
- **Search runs where the data lives.** Glob and grep are backend primitives
  precisely so a remote backend can execute them server-side instead of
  paging the tree through the agent.
- **Paths are workspace-relative**, normalized before the backend sees them;
  escapes (symlinks, `..`) are rejected uniformly; depth and segment limits
  apply to every operation.
- **Snapshots are opaque references.** The token means something only to the
  backend that minted it. Restore is all-or-nothing and MUST return the
  workspace to the exact observed state — including files that ignore rules
  would normally hide, and removing files created after the snapshot. A
  restore that "mostly" rolls back breaks the tool transaction contract.

### The command facet: argv, not shell

Command execution takes an argument vector and executes it directly — no
shell interpretation, no globbing, no variable expansion. Working directories
are workspace-relative and escape-checked. Output is capped with explicit
truncation flags; timeouts are mandatory with sane defaults; launch failures
map to conventional exit codes rather than exceptions, so the model always
gets a structured result it can reason about. The base environment is
constructed hygienically (host environment filtered), with declared variables
layered on top.

The rationale is the same as for typed tool parameters: shell strings are an
injection surface and an ambiguity generator, and an unattended agent
composing shell strings from model output is a known incident category.

### Egress and credentials as a control plane

- **Default deny.** An empty egress policy allows nothing. Rules are explicit
  allows: host pattern, ports, protocol, optional credential to inject.
- **Credentials by name.** Definitions and configurations carry credential
  *names*. Material is bound at run time by the operator side, injected only
  at the enforcement point (e.g., a proxy adding a header), never written
  into the workspace, the environment variables, logs, or serialized state,
  and cleared on close.
- **Not a tool surface.** Reconfiguring egress or credentials is available to
  the operator and control plane, never to the agent through its tool
  context. An agent MUST NOT be able to widen its own reach — this single
  sentence is most of the prompt-injection blast-radius story.

### Rendering from reality

Anything the definition shows the model about the workspace (a file listing,
a preview) MUST be derived from the materialized environment at render time —
not from copy-time bookkeeping that goes stale the moment setup commands run.
Before materialization, previews render an explicit placeholder rather than a
guess.

## Anti-patterns

- **Host paths in definitions.** Any absolute host path in instructions or
  tool parameters is a portability bug and usually a security bug.
- **Shell-string convenience.** One `sh -c` escape hatch reintroduces every
  problem the argv contract removed.
- **Secrets in intent.** A credential value in a config file is a leak with a
  commit history.
- **Partial rollback.** Backends that restore tracked files but leave
  untracked debris make retries unsafe in exactly the way that is hardest to
  debug.

## Compatibility surface

An adapter certification suite MUST assert:

- The same declared intent materializes to equivalent observable workspaces
  (mounts present, setup applied, posture enforced) on every harness.
- Filesystem semantics (pagination, write modes, escape rejection, limits)
  are identical over every backend via the shared validation suite.
- Snapshot → restore returns the workspace to the exact prior state.
- Egress default-deny holds: undeclared destinations are unreachable from the
  workspace.
- Secret material appears nowhere the agent or the run record can read.
