# Workspaces

The workspace is the remote filesystem the agent uses while it works. The
harness reads and writes files there, long-running work leaves outputs there,
and future turns resume from it after compute restarts.

The workspace is where the agent works. It is not necessarily where the source
of truth lives. Analytical systems may live in warehouses, object stores,
feature stores, notebooks, dashboards, application databases, or external APIs.

## Model

A production workspace is remote, tenant-scoped, and mounted near the harness.
It is not a temporary directory on the application host.

The workspace survives:

- harness process restart
- sandbox compute restart
- harness adapter disconnect
- application deployment
- turn retry or reattach

It does not survive explicit deletion, tenant teardown, retention expiry, or
workspace cleanup commands.

Some sandbox filesystems last only as long as active compute. That is still
useful, but it is not the same as a persistent workspace. A production design
needs either a durable workspace mounted near the harness, or a protocol that
can rehydrate the sandbox from remote storage before work resumes.

## Path Rules

The public protocol uses sandbox-relative paths. The sandbox validates paths and
prevents traversal outside the workspace. The application does not ask for host
paths such as `/etc/passwd`, and does not expect sandbox paths to exist locally.

Useful properties:

- path-addressed operations
- no cross-call file handles
- streaming reads and writes
- content type and size metadata for uploads
- explicit binary and text APIs
- predictable overwrite behavior
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

Durable workspace data and scratch data are different.

Durable data includes source checkout, outputs, persistent notes, generated
files, query result references, reports, and work-in-progress that future turns
need.

Scratch data includes temporary extraction directories, transient tool outputs,
and partial files from failed operations. Scratch data is tied to a turn,
transaction, or retention policy.

The sandbox owns cleanup of sandbox-local state. Runtime-facing integration
requests cleanup through the protocol, not by unlinking sandbox paths directly.

## Snapshots

Snapshots are protocol operations. The sandbox returns opaque snapshot tokens;
the application does not inspect storage internals.

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

Workspace identity combines tenant, sandbox, repository or project, and durable
work identity. The exact tuple is platform-specific, but the required property is
simple: two tenants must not collide, and two independent units of work do not
accidentally share mutable files.

Shared workspaces are allowed when explicitly declared. Sharing is part of the
work contract, not an accident of path reuse.

For analytical work, workspace identity is not a substitute for data lineage. A
run record still needs to identify which data source, schema, metric version,
query execution, or result handle supported the final output.

## Skills and Runtime Files

Skill bundles, helper scripts, and runtime files are uploaded or mounted through
the same workspace-aware protocol. The sandbox does not assume it can read them
from the application's local disk.
