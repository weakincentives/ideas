# Evaluation and Conformance

Agent definition libraries need evaluation support because prompt and tool
design are empirical. Iteration still needs rigor.

## Agent Loop

An agent loop runs one unit of work from definition to result. It:

- accepts caller-owned work identifiers
- renders the definition
- binds resources
- starts or reconnects durable work
- streams events
- fulfills definition tools
- enforces budgets and deadlines
- runs completion checks
- returns typed output or typed terminal error

The loop is application orchestration. The harness still owns the model act
loop.

## Eval Loop

An eval loop runs many agent-loop instances against a dataset. It tracks:

- dataset version
- definition version
- prompt override set
- driver and harness versions
- model settings
- skill versions
- workspace seed
- evaluator versions
- random seeds or time injection settings
- result metrics

Evaluations are repeatable enough to compare changes while accepting that model
behavior may still vary.

## Datasets

Datasets are versioned assets, not loose collections of prompts. Each case
defines:

- input
- expected output or rubric
- workspace seed files
- allowed tools
- completion criteria
- evaluator configuration
- tags and metadata

Workspace seeds are staged through the same workspace protocol used in
production. Local fixture directories are only source material.

## Evaluators

Evaluators are typed functions over run artifacts. They may inspect final
output, event history, workspace state, tool calls, or debug bundle data.

LLM-as-judge evaluators record their own model, prompt, rubric, thresholds, and
raw output. They can help, but they are not inherently authoritative.

## Prompt Overrides

Prompt overrides let teams test changes without changing source definitions.
They are hash-validated and recorded in evaluation metadata.

An override that no longer applies because the base prompt changed fails closed.
Silent fuzzy matching creates misleading evals.

## Reporting

Eval reports include:

- aggregate metrics
- per-case failures
- regression deltas
- tool-call and native-event summaries
- budget summaries
- representative transcripts
- links or references to debug bundles
- driver conformance failures

The report makes it obvious whether a regression came from definition changes,
driver behavior, skill changes, harness behavior, model behavior, or sandbox
infrastructure.
