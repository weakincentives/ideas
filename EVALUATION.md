# Evaluation and Contract Tests

Evaluation is a support surface. It does not define application intent and
it does not run the harness directly. It compares completed or repeatable
work using definitions, datasets, run records, and contract tests.

For unattended analytical agents, evaluation has to handle a harder question
than "is the output correct". It also has to tell teams whether worse
behavior came from application-facing integration, the definition,
runtime-facing integration, the skills, the harness, the model, or the
sandbox. Without that attribution, evals become noisy and teams stop
trusting them.

## Eval Loop

An eval loop runs agent work against a dataset. It tracks:

- dataset version
- definition version
- prompt override set
- harness adapter and harness versions
- model settings
- skill versions
- workspace seed
- evaluator versions
- random seeds or time injection settings
- data source fixture or metric versions
- result metrics

Evaluations should be repeatable enough to compare changes while accepting
that model behavior may still vary.

## Datasets

Datasets are versioned inputs, not loose collections of prompts. Each case
defines:

- input
- expected output or rubric
- workspace seed files
- data source fixtures or references
- allowed tools
- completion criteria
- evaluator configuration
- tags and metadata

Workspace seeds are staged through the same workspace protocol used in
production (see [WORKSPACES.md](WORKSPACES.md)). Local fixture directories
are source material, not a production path.

## Evaluators

Evaluators are typed functions over run records. They may inspect final
output, event history, workspace state, tool calls, query results,
generated outputs, or debug bundle data. They read the records described in
[RUN-RECORDS.md](RUN-RECORDS.md).

Analytical evaluators may check known-answer cases, synthetic datasets,
query result validation, statistical sanity checks, reproducibility,
lineage, and cost or performance budgets.

Analytical judgment is not always a single right answer. Evals should
separate factual correctness, query correctness, data sufficiency,
reasoning quality, causal claims, business judgment, and presentation
quality. For example, a good outcome on a case may be "the data is
insufficient to answer." An eval that forces a confident answer in that
case teaches the agent to fabricate.

LLM-as-judge evaluators record their own model, prompt, rubric,
thresholds, and raw output. They can help, but they do not get special
authority.

## Contract Tests

Contract tests verify that an integration layer implements the shared
behavior. They cover both integration directions.

Application-facing tests cover:

- binding application tools
- binding policies and resources
- query engines and data tools
- retry behavior
- eval fixtures
- output consumers
- tool call completion
- policy denial
- feedback delivery
- structured output validation
- review and delivery rules

Runtime-facing tests cover:

- definition rendering for a harness
- tool schema rendering
- skill packaging
- built-in event translation
- supported transaction behavior
- workspace upload and read/write
- duplicate starts
- reconnect
- disconnect grace expiry
- debug bundle generation

Contract tests are what let a team honestly claim that the same definition
runs against more than one harness or sandbox. Without them, cross-harness
support is a slogan. The readiness conditions they enforce are stated in
[DEFINITION.md](DEFINITION.md).

## Prompt Overrides

Prompt overrides let teams test changes without changing source
definitions. They are hash-validated and recorded in evaluation metadata.
An override that no longer applies because the base prompt changed fails
closed. Silent fuzzy matching creates misleading evals.

## Reporting

Eval reports include aggregate metrics, per-case failures, before/after
deltas, tool-call and built-in event summaries, budget summaries,
representative transcripts, query summaries, data lineage, validation
checks, links or references to debug bundles, and contract test failures.

The report should make it clear whether worse behavior came from
application-facing integration, definition changes, runtime-facing
integration, skill changes, harness behavior, model behavior, or sandbox
infrastructure. Attribution is the whole point.
