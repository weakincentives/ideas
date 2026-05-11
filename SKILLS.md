# Skills and Runtime Assets

A skill packages operating knowledge for a model-harness runtime. It teaches the
harness how to use a repository, workflow, data source, domain, or tool
correctly.

Skills are runtime-facing. They help the harness operate; they are not a place
to hide application side effects or business logic.

## Why Skills Matter

When a model is used through a harness, the shape of the context matters. A
harness may have its own rules for finding skills, mounting files, reading
manifests, running helper scripts, and showing examples.

Good harness adapters feed those rules. They do not flatten every skill into a
generic prompt block.

Skills are one of the main places application teams spend effort. They turn
local operating knowledge into reusable, testable harness input: repository
conventions, metric definitions, safe query patterns, reporting expectations,
and validation habits.

## Skill Contents

A skill may contain:

- instructions
- examples
- scripts
- schemas
- tool usage notes
- repository conventions
- data source conventions
- metric definitions
- safe query patterns
- validation workflows
- reporting conventions
- troubleshooting notes
- test fixtures
- manifest metadata
- references to required definition tools
- runtime files that must be uploaded to the sandbox

The packaging format is harness-specific. The important requirement is that the
harness adapter can stage, mount, version, observe, and test the skill.

## Skills and Tools

Skills explain how to operate. Tools perform application and data-system
actions.

A skill can tell the harness when and how to call a definition tool. The tool
still needs a typed schema, policy, result shape, and idempotency behavior.

A skill can explain how to inspect data, avoid expensive queries, validate
joins, compare cohorts, or interpret a metric. The query still runs through a
declared query tool with permissions, budgets, and records.

A skill can include scripts or helper files. If those scripts mutate the
workspace or call the network, the harness adapter and sandbox policy make that
visible.

## Remote Staging

In production, the harness does not read skills from the caller's local disk.
Skills are staged into the remote sandbox or referenced through a protocol the
sandbox can resolve.

Skill staging records:

- skill name
- version or content hash
- source reference
- files staged
- target mount path
- required tools
- required permissions
- harness adapter version
- harness version

This makes a run explainable after the local checkout has moved on.

## Evaluation

Skills are tested with runtime-facing integration. Tests cover:

- discovery
- staging
- rendering or mounting
- tool references
- missing dependency behavior
- sandbox path assumptions
- representative task performance

Treat skill failures as integration failures, not prompt polish.

## Anti-Patterns

- embedding all operating knowledge in one giant prompt
- relying on the harness to read local files directly
- shipping scripts whose side effects bypass sandbox policy
- using skills to hide undeclared tools
- using skills to hide data access
- keeping skills outside version control
- evaluating definitions without the skills used in production
