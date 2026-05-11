# Skills and Runtime Assets

A skill packages operating knowledge for a model-harness runtime. It teaches
the harness how to use a repository, workflow, data source, domain, or tool
correctly. Skills are runtime-facing: they help the harness operate. They are
not a place to hide application side effects, data access, or business
logic.

Where tools perform actions, skills explain how to act. A team can declare a
well-shaped data tool and still produce wrong results if the agent does not
know which joins are usually wrong, which filters always apply, and which
metric version is current. Skills capture that knowledge in a form the
harness can load, stage, and version.

## Why Skills Matter

When a model is used through a harness, the shape of the context matters. A
harness may have its own rules for finding skills, mounting files, reading
manifests, running helper scripts, and showing examples. Good harness
adapters feed those rules. They do not flatten every skill into a generic
prompt block.

Skills are one of the places application teams spend real effort. They turn
local operating knowledge into reusable, testable harness input: repository
conventions, safe query patterns, metric definitions, reporting
expectations, and validation habits.

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

For example, a "weekly cohort report" skill might carry the metric owner,
the canonical join between user and subscription tables, the privacy filter
that always applies, the warehouse session settings that affect percentile
functions, and a handful of example queries with expected row shapes. The
skill does not perform the query; it teaches the harness how to write a
correct one and when to call the `run_warehouse_query` tool.

The packaging format is harness-specific. The important requirement is that
the harness adapter can stage, mount, version, observe, and test the skill.

## Skills and Tools

Skills explain how to operate. Tools perform application and data-system
actions. A skill can tell the harness when and how to call a definition
tool; the tool still needs a typed schema, policy, result shape, retry
behavior, and records. See [CAPABILITIES.md](CAPABILITIES.md) for the tool
shape.

A skill can explain how to inspect data, avoid expensive queries, validate
joins, compare cohorts, or interpret a metric. The query still runs through
a declared query tool with permissions, budgets, and records.

A skill can include scripts or helper files. If those scripts mutate the
workspace or call the network, the harness adapter and sandbox policy make
that visible rather than hide it.

## Data Knowledge

Analytical skills often carry data knowledge:

- which metric definition is current
- which owner understands a table
- which freshness rule matters for a report
- which joins are usually wrong
- which privacy filters always apply
- which warehouse settings affect results

That knowledge helps the agent operate. It does not replace records. Tools
and run records still need to capture which metric, schema, data source, or
skill version was used. A skill is a teacher, not a witness.

## Remote Staging

In production, the harness does not read skills from the caller's local
disk. Skills are staged into the remote sandbox or referenced through a
protocol the sandbox can resolve. See [WORKSPACES.md](WORKSPACES.md) for the
staging mechanics.

Skill staging records include:

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

Skills are tested through runtime-facing integration, alongside the
definition and harness they run under. Tests cover discovery, staging,
rendering or mounting, tool references, missing-dependency behavior, sandbox
path assumptions, and representative task performance. See
[EVALUATION.md](EVALUATION.md) for how contract tests exercise both
integration directions.

Treat skill failures as integration failures, not prompt polish.

## Anti-Patterns

- embedding all operating knowledge in one giant prompt
- relying on the harness to read local files directly
- shipping scripts whose side effects bypass sandbox policy
- using skills to hide undeclared tools
- using skills to hide data access
- keeping skills outside version control
- evaluating definitions without the skills used in production
