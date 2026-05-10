# Skills and Runtime Assets

Skills are versioned integration assets for execution substrates. They package
the operational knowledge a harness needs in order to use a repository,
workflow, domain, or tool correctly.

## Why Skills Matter

If the model and harness are trained and evaluated together, the shape of the
context matters. A harness may have conventions for skill discovery, file
layout, manifests, scripts, examples, tool descriptions, and runtime assets.
Good drivers feed those conventions rather than flatten every harness into
generic prompt text.

Skills are one of the main places application teams spend effort. They turn
local operational knowledge into reusable, testable harness input.

## Skill Contents

A skill may contain:

- instructions
- examples
- scripts
- schemas
- tool usage notes
- repository conventions
- troubleshooting notes
- test fixtures
- manifest metadata
- references to required definition tools
- runtime assets that must be uploaded to the sandbox

The exact packaging format is harness-specific. The design requirement is that
the driver can stage, mount, version, observe, and test the skill.

## Skill Boundary

Skills explain how to operate. Tools perform application side effects.

A skill can tell the harness when and how to call a tool. The tool still needs a
typed schema, policy, result contract, and idempotency behavior.

A skill can include scripts or helper files. If those scripts mutate the
workspace or call the network, the driver and sandbox policy make that visible.

## Remote Staging

In production, skills are not read from the caller's local disk by the harness.
They are staged into the remote sandbox or referenced through a protocol the
sandbox can resolve.

Skill staging records:

- skill name
- version or content hash
- source reference
- files staged
- target mount path
- required tools
- required permissions
- driver and harness version

This makes a run explainable after the local checkout has moved on.

## Evaluation

Skills are tested as part of the driver conformance suite. Tests cover:

- discovery
- staging
- rendering or mounting
- tool references
- missing dependency behavior
- sandbox path assumptions
- representative task performance

Treat skill regressions as integration regressions, not prompt polish.

## Anti-Patterns

- embedding all operating knowledge in one giant prompt
- relying on the harness to read local files directly
- shipping scripts whose side effects bypass sandbox policy
- using skills to hide undeclared tools
- keeping skills outside version control
- evaluating definitions without the skills used in production
