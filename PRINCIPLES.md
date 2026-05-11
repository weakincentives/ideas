# Principles

These rules govern the rest of the corpus. They are written for unattended
background agents that do analytical work: reading repositories, writing code,
running queries, producing durable outputs, and leaving evidence that other
people can review. Treat them as claims, not options. Every other chapter in
this repository should be readable as a consequence of the rules below.

## 1. Treat the Model and Harness as One Runtime

The stable target is not a bare model API. Useful behavior comes from the model
as used through a harness: tool shape, edit loops, filesystem assumptions,
approval modes, message formats, skill loading, scheduling, transcripts, and
recovery. The same model behind a raw chat API is a different system with
different failure modes.

An agent with file search, patch application, shell execution, query tools,
approval flow, and transcript handling is not equivalent to a model that only
returns text. Design for the runtime the agent actually uses.

## 2. Do Not Rebuild the Harness Loop

The harness already owns the model loop, built-in tools, provider integration,
and runtime recovery. A definition library should drive that stack, connect it
to application systems, and record what it does. Rebuilding the loop from
scratch throws away the harness's recovery behavior and the tool shape that
made the runtime useful.

Application teams still own the gates around unattended work: cost limits,
data access, freshness checks, acceptance checks, escalation, review, and
delivery.

## 3. Keep the Definition in the Middle

The agent definition is the contract between application intent and runtime
behavior. It is not the application, and it is not the harness. Application
systems bind into it through application-facing integration. Harnesses run it
through runtime-facing integration. Keeping the definition stable is what lets
the same intent survive new harnesses, sandboxes, or deployment models.

## 4. Keep Instructions Near Capabilities

The prompt is structured input, not a wall of text. It tells the agent what it
can see, what it can do, which rules apply, and what output is expected.
Related content belongs together: a tool appears near the instructions that
teach the agent how to use it, a policy appears near the behavior it
constrains, a completion check appears near the output it validates.

Structure also makes the definition inspectable later, which matters once a
run is being reviewed.

## 5. Use Tools for Actions and Skills for Know-How

Application and data-system actions go through declared tools, not broad
sandbox network access. Every action has a name, a typed input, a typed
output, a policy check, a retry story, and a trace. Skills package operating
knowledge: how to use a repository, workflow, domain, data source, or tool
correctly.

Skills do not replace tools. Tools do not replace skills. A skill can teach
the harness when to call a tool; the tool still performs the action and leaves
records.

## 6. Make Data Meaning Explicit

A metric is not a table name. Analytical definitions need business meaning,
filters, freshness rules, ownership, privacy limits, and data limits that make
a result interpretable. The agent should not have to infer important meaning
from naming conventions.

When meaning is left implicit, the agent will confidently produce numbers that
nobody can defend.

## 7. Design for Remote Sandboxes First

Assume the harness and filesystem run in a separate sandbox reached through a
protocol. Shared memory, in-process callbacks, and local paths are development
conveniences, not design premises. Local mode is the remote design in a
simpler setting, not a second design.

Remote-first discipline forces the real boundaries into view early: work
identity, tool routing, workspace persistence, data access, and egress
control.

## 8. Make Retries Safe

Durable work needs stable public identifiers supplied by the caller. Backend
session IDs, provider request handles, and runtime trace tokens can exist, but
they are not a substitute for an identifier the caller can retry with.

Starting the same work twice should reconnect to the existing work or return a
clear conflict. Silent double-execution is a correctness bug, not a
performance issue.

## 9. Keep Work, Compute, and Connections Separate

A connection is not work. A compute process is not work. Disconnecting does
not cancel work by default. Reconnecting does not start new work. A sandbox
process can restart without deleting the workspace or the caller's work
record.

Collapsing these lifecycles into one is the single most common source of
accidental lost work and accidental duplicate work.

## 10. Be Honest About Rollback

Transactions only cover state the definition library controls. They do not
roll back an external API call that already happened, an unwrapped built-in
harness command, or a filesystem mutation outside a sandbox snapshot.

When rollback is not available, say so and record the mutation. A polished
"transaction aborted" message that hides a real external side effect is worse
than a visible failure.

## 11. Leave Records

Every run leaves enough evidence to reconstruct what happened: rendered
definition, tool schemas, tool calls, query records, freshness checks, review
decisions, built-in harness events, workspace references, outputs, budgets,
errors, and trace links. A run that cannot be reconstructed was not observed
in any useful sense.

Records should explain the run without embedding sensitive raw data
unnecessarily. References, redaction, and retention rules are part of record
design, not an afterthought.

## 12. Prove Integration Behavior

Cross-harness support is not a slogan. An integration layer passes shared
contract tests before claiming that the same definition can run against more
than one harness or sandbox. Differences between harnesses belong in adapters
and are made visible, not flattened.

A harness adapter should not fake unsupported behavior. If a required
capability is missing, the adapter reports the limit and the test proves it.
