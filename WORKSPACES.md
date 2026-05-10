# Workspaces

The workspace is the agent's durable data plane. It is where the harness reads
and writes files, where long-running work leaves outputs, and where future turns
can resume context after compute restarts.

## Model

A production workspace is remote, tenant-scoped, and mounted near the harness.
It is not a temporary directory on the orchestrator host.

The workspace survives:

- harness process restart
- sandbox compute restart
- harness adapter disconnect
- orchestrator deployment
- turn retry or reattach

It does not survive explicit deletion, tenant teardown, retention expiry, or
workspace cleanup commands.

## Path Rules

The public protocol uses sandbox-relative paths. The sandbox validates paths and
prevents traversal outside the workspace. The orchestrator does not ask for host
paths such as `/etc/passwd`, and does not expect sandbox paths to exist locally.

Recommended properties:

- path-addressed operations
- no cross-call file handles
- streaming reads and writes
- content-type and size metadata for uploads
- explicit binary and text APIs
- deterministic overwrite behavior
- typed errors for missing path, conflict, permission denial, and quota

## Staging

Initial files are staged into the workspace before a turn begins. Staging is
explicit:

- caller declares allowed host roots
- caller uploads files through the protocol
- sandbox stores them under the workspace prefix
- staged files are recorded in the work contract or workspace manifest

For large files, the protocol supports upload tickets or signed upload URLs so
the data path does not require buffering through the control channel.

## Persistence and Cleanup

Durable workspace data and evaluation-scoped scratch data are different.

Durable data includes source checkout, outputs, persistent notes, generated
files, and work-in-progress that future turns need.

Scratch data includes temporary extraction directories, transient tool outputs,
and partial files from failed operations. Scratch data is tied to a turn,
transaction, or retention policy.

The sandbox owns cleanup of sandbox-local state. The orchestrator requests
cleanup through the protocol, not by unlinking sandbox paths directly.

## Snapshots

Snapshots are protocol operations. The sandbox returns opaque snapshot tokens;
the orchestrator does not inspect storage internals.

Snapshot operations define:

- scope
- lifetime
- quota impact
- commit behavior
- rollback behavior
- timeout cleanup
- error behavior if rollback cannot complete

Snapshot and restore are required for strong filesystem transaction guarantees.
Without them, the definition library can still observe filesystem mutations but
cannot honestly claim rollback.

## Workspace Identity

Workspace identity composes tenant, sandbox, repository or project, and
caller-owned work identifiers. The exact tuple is platform-specific, but the
property is not: two tenants must not collide, and two independent units of work
do not accidentally share mutable files.

Shared workspaces are allowed when explicitly declared. Sharing is part of the
contract, not an accident of path reuse.

## Skills and Runtime Assets

Skill bundles, helper scripts, and runtime assets are uploaded or mounted
through the same workspace-aware protocol. The sandbox does not assume it can
read them from the orchestrator's local disk.
