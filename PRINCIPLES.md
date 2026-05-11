# Principles

These rules guide the rest of the repository. They are written for unattended
background agents doing analytical work: code, queries, files, long-running
tasks, and durable outputs.

## 1. Keep the Model and Harness Together

The stable target is not a bare model API. Useful behavior comes from the model
as used through a harness: tools, edit loops, filesystem shape, approval modes,
message formats, skills, scheduling, and recovery behavior.

A background agent with file search, patch application, shell execution, query
tools, approval flow, skill loading, and transcript handling is not equivalent
to the same model behind a raw chat API. The tool shape changes what the model
can do, what the harness can recover from, and what evidence remains.

## 2. Do Not Rebuild the Harness Loop

The harness already owns the model loop, built-in tools, provider integration,
and runtime recovery. A definition library should drive that stack, connect it
to application systems, and record what it does.

Application teams still own the gates around unattended work: cost limits, data
access, freshness checks, acceptance checks, escalation, review, and delivery.

## 3. Keep the Definition in the Middle

The definition is the contract between application intent and runtime behavior.
It is not the application, and it is not the harness.

Application-facing integration binds product data, query engines, tools,
policies, resources, outputs, and eval fixtures into the definition.

Runtime-facing integration renders the definition into one harness, stages
skills, connects workspaces, routes tool calls, streams events, and handles
runtime differences through harness adapters and sandbox protocols.

## 4. Keep Instructions Near Capabilities

The prompt is structured input, not just text. It tells the agent what it can
see, what it can do, which rules apply, and what output is expected.

A tool appears near the instructions that teach the agent how to use it. A
policy appears near the behavior it constrains. A completion check appears near
the output it validates.

## 5. Use Tools for Actions and Skills for Know-How

Application and data-system actions go through declared tools, not broad network
access from inside the sandbox. Every action has a name, typed input, typed
output, policy check, retry story, and trace.

Skills package operating knowledge: how to use a repository, workflow, domain,
data source, query engine, or tool correctly. Skills do not replace tools, and
tools do not replace skills.

## 6. Make Data Meaning Explicit

A metric is not just a table name. Analytical definitions need the business
meaning, filters, freshness rules, ownership, privacy limits, and data limits
that make a result interpretable.

The agent should not have to infer important meaning from naming conventions.

## 7. Design for Remote Sandboxes First

Assume the harness and filesystem run in a separate sandbox reached through a
protocol. Shared memory, in-process callbacks, and local paths are development
conveniences, not design premises.

## 8. Make Retries Safe

Durable work needs stable caller-supplied identifiers. Backend session IDs,
provider request handles, and runtime trace tokens can exist, but they are not a
substitute for a public identifier the caller can retry with.

Starting the same work twice should reconnect to the same work or return a
clear conflict. It should not silently double-execute.

## 9. Keep Work, Compute, and Connections Separate

A connection is not work. Disconnecting does not cancel work by default.
Reconnecting does not start new work. A sandbox process can restart without
deleting the workspace or the caller's work record.

## 10. Be Honest About Rollback

Transactions only cover state the definition library controls. They do not roll
back an external API call that already happened, an unwrapped built-in harness
command, or a filesystem mutation outside a sandbox snapshot.

## 11. Leave Records

Every run leaves enough evidence to reconstruct what happened: rendered
definition, tool schemas, tool calls, query records, freshness checks, review
decisions, built-in harness events, workspace references, outputs, budgets,
errors, and trace links.

Records should explain the run without embedding sensitive raw data
unnecessarily.

## 12. Prove Integration Behavior

Cross-harness support is not a slogan. An integration layer passes shared
contract tests before claiming that the same definition can run against more
than one harness or sandbox.
