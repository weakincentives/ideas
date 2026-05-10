# Evaluation and Conformance

Agent definition libraries need evaluation support because prompt and tool
design are empirical. The goal is to iterate without losing rigor.

## Agent Loop

An agent loop runs one unit of work from definition to result. It should:

- accept caller-owned work identifiers
- render the definition
- bind resources
- start or reconnect durable work
- stream events
- fulfill definition tools
- enforce budgets and deadlines
- run completion checks
- return typed output or typed terminal error

The loop is application orchestration. The harness still owns the model act
loop.

## Eval Loop

An eval loop runs many agent-loop instances against a dataset. It should track:

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

Evaluations should be repeatable enough to compare changes, while accepting
that model behavior may still vary.

## Datasets

Datasets are versioned assets, not loose collections of prompts. Each case
should define:

- input
- expected output or rubric
- workspace seed files
- allowed tools
- completion criteria
- evaluator configuration
- tags and metadata

Workspace seeds should be staged through the same workspace protocol used in
production. Local fixture directories are acceptable only as source material.

## Evaluators

Evaluators should be typed functions over run artifacts. They may inspect final
output, event history, workspace state, tool calls, or debug bundle data.

LLM-as-judge evaluators should record their own model, prompt, rubric,
thresholds, and raw output. They are useful but not inherently authoritative.

## Prompt Overrides

Prompt overrides let teams test changes without changing source definitions.
They should be hash-validated and recorded in evaluation metadata.

An override that no longer applies because the base prompt changed should fail
closed. Silent fuzzy matching creates misleading evals.

## Reporting

Eval reports should include:

- aggregate metrics
- per-case failures
- regression deltas
- tool-call and native-event summaries
- budget summaries
- representative transcripts
- links or references to debug bundles
- driver conformance failures

The report should make it obvious whether a regression came from definition
changes, driver behavior, skill changes, harness behavior, model behavior, or
sandbox infrastructure.
