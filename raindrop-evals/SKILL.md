---
name: raindrop-evals
description: Create, discover, and run Raindrop evals against a local application using Vitest or the JavaScript SDK. Reuse UI-created datasets and evals, preserve reference answers, and leave a repeatable run with saved results.
---

Connect the user's actual application to repeatable evals. Handle discovery,
setup, execution, and result inspection; the user should not have to find slugs.
Read [the local workflow](references/local-workflow.md) for tested SDK calls.

## Find the inputs

Identify the application's entry point, tracing integration, package manager,
and Raindrop project. Use the existing environment for credentials. The Query
API key and application tracing must target the same organization and project.

Use MCP `raindrop_skills({ topic: "evals" })` for the current evaluator authoring
contract. `list_datasets({ project })` returns IDs, slugs, and row counts;
`list_datasets({ project, dataset_id: idOrSlug })` shows rows and reference answers.
`list_evals({ project })` returns saved evaluator slugs, output types, and execution
modes. Inspect the selected evaluator with `get_eval({ project, eval: slug })`.
UI-created and agent-created resources share these identities. Reuse them.

Inspect representative rows. The input goes to the application. Optional `output`
is the reference answer for grading; `referenceTrace` is separate optional evidence.
`expectedVerdict` labels a calibration example's expected grade. Do not put either
the reference answer or expected grade into the application prompt, and do not
fabricate traces from plain answers.

## Create an eval when needed

Use deterministic code for objective checks and a judge for semantic criteria.
Choose boolean, score (1–5), or number output. Load the current MCP authoring
instructions before writing program source; programs emit with `ctx.grade`.

Create a labeled calibration dataset from real captured examples, including good
and bad outputs. MCP `publish_eval_dataset` accepts source event IDs; poll
`get_eval_dataset_publication` when the publication is queued. Include timestamps
for older events. Keep calibration labels independently justified; do not derive
expected grades from the evaluator being tested.

Call `create_eval({ project, name, intent, program_source, reference_dataset,
execution_mode, output_type })`. `reference_dataset` is the calibration dataset's
slug or ID. Read `get_eval` until validation completes and inspect disagreements
and errors. Revise with `update_eval_program` when needed. Calibration has its own
validation record; don't create ordinary eval runs merely to smoke-test a grader.

Plain reference-answer datasets work for application runs. MCP calibration needs
captured outputs with expected verdicts: input/reference-answer rows alone do not
provide an actual answer to grade. When those examples do not exist yet, run the
application and inspect its captured outputs before selecting calibration labels.

## Save and run

Save a JSON manifest with a dataset ID/slug, one or more evaluator slugs, and
thresholds appropriate to their output types. Bind it to the real instrumented
application using `defineEvalSuite(manifest, { agent })`. The agent definition owns
setup and cleanup, including flushing telemetry. Omit version pins unless the
user asks for a specific version; the run records resolved versions.

Use `evalTests(suite)` for Vitest or the SDK/CLI runner for direct execution.
Saved evaluator slugs grade in Raindrop. `defineLocalEvaluator` callbacks grade
in the local process and their results are uploaded. A discovered local evaluator
registration contains its identity, not its executable callback; import that code
from the repository. No separate local Raindrop service is needed.

Check that every selected row completed and every evaluator produced a grade.
Zero grades, missing traces, or execution errors are setup failures. Keep them
separate from an answer failing a quality threshold. For initial setup, verify a
known bad example fails without changing the user's reference data or lowering
the standard. Save the rerun command and return the Raindrop results links with
a short explanation of the failures. A synthetic setup check proves the plumbing,
not the quality of the customer's application.
