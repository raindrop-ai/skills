# Local run setup

Use the supplied Raindrop release packages or the project's pinned versions.
This workflow requires `EvalSuiteManifestSchema` and the two-argument
`defineEvalSuite` API. Check the installed exports before setup; older SDK versions
may silently omit reference answers. Do not assume npm `latest` includes a change
that has only been supplied as a package artifact. Install `raindrop-ai`,
`@raindrop-ai/vitest`, Vitest, and the tracing integration for the user's framework
from the agreed release. Keep the lockfile.

Credentials: `RAINDROP_QUERY_API_KEY` and `RAINDROP_PROJECT_ID` for evals, plus the
application's existing tracing/model credentials. Load them through the project's
existing env mechanism. Never embed credentials in a manifest or committed file.

## Existing datasets

After MCP discovery, verify the selected dataset through the SDK:

```js
import Raindrop, { readEvalDataset } from "raindrop-ai";
const client = new Raindrop({
  apiKey: process.env.RAINDROP_QUERY_API_KEY,
  projectId: process.env.RAINDROP_PROJECT_ID,
  localWorkshopUrl: false,
});
try {
  const dataset = await readEvalDataset(client, selectedDatasetIdOrSlug);
  // dataset.rows contains input, output, properties, and optional trace references.
  // dataset.version.id identifies this immutable snapshot.
} finally {
  await client.close();
}
```

Existing UI datasets retain their IDs. Run-time snapshots preserve earlier data
when the user later edits the dataset. Don't copy it into a new dataset per eval.
Dataset versions currently support up to 1,000 rows and publication requests up to
4 MiB. Check size before setup; never silently truncate an existing dataset.

To upload new reference-answer rows:

```js
await publishEvalDataset(client, {
  slug: "lead-research",
  name: "Lead research",
  rows: [{
    id: "company-name", name: "Company name",
    input: "What is the company name in this supplied record? ...",
    output: "Acme Labs", properties: {},
  }],
});
```

Import `publishEvalDataset` from `raindrop-ai`. `output` can also be JSON, or omitted.
When deliberately changing an existing published dataset, pass its discovered
`expectedCurrentVersionId`; a conflict means someone else edited it. Read again
before deciding to overwrite. SDK-generated request keys handle publication retries.
For inline datasets use `defineDataset({ id: "lead-research", name, rows })`;
`id` becomes its publication slug. Selected tests publish these rows automatically.

## One manifest, multiple evals

`evals/quality.json`:

```json
{
  "name": "Lead research quality",
  "dataset": "lead-research",
  "evaluators": [
    { "evaluator": "correctness", "threshold": { "equals": true } },
    { "evaluator": "conciseness", "threshold": { "gte": 4 } }
  ]
}
```

Replace the illustrative slugs with discovered ones and match thresholds to their
output types. Boolean thresholds use `equals`; score/number thresholds use `gte`
or `lte`. Omitted thresholds record grades without a quality gate. The UI can
create this manifest: eval → New run → Replay agent → Locally with Vitest or SDK →
choose dataset → Copy manifest. Add more discovered evals when requested.

Bind the application in `evals/quality.suite.mjs`:

```js
import { readFileSync } from "node:fs";
import { defineAgent, defineEvalSuite } from "raindrop-ai";
import { runAgent, closeTelemetry } from "../src/agent.js";

const manifest = JSON.parse(readFileSync(new URL("quality.json", import.meta.url), "utf8"));
const agent = defineAgent({
  slug: "lead-research",
  run: ({ input }) => runAgent(input),
  cleanup: () => closeTelemetry(),
});
export default defineEvalSuite(manifest, { agent });
```

Adapt the imports and callback to the real repository. `runAgent` must use the
existing Raindrop tracing integration; the runner associates those spans with
the dataset row. Flush telemetry before cleanup. If the app needs shared resources,
use `defineAgent`'s `setup`, `run(row, { environment })`, and `cleanup(environment)`.

`evals/quality.test.mjs`:

```js
import { evalTests } from "@raindrop-ai/vitest";
import suite from "./quality.suite.mjs";
await evalTests(suite);
```

Run either command with the same environment:

```sh
node --env-file=.env ./node_modules/vitest/vitest.mjs run evals/quality.test.mjs
node --env-file=.env ./node_modules/raindrop-ai/dist/evals/cli.mjs run evals/quality.suite.mjs
```

Vitest discovery does not publish a dataset or create a run. Execution checks the
collected rows still match. Hosted graders run as one batch test; all-local callback
graders get row tests. Both fail on errors and on configured threshold violations.
Direct code can use `runEvalSuite(client, suite)`; inspect `result.rows` and
`result.evaluators`, close resources, and set a failing exit code for failed checks.

## Local grading

Keep the callback in code and pass its object in a normal suite definition:

```js
const matchesReference = defineLocalEvaluator({
  slug: "exact-answer", name: "Exact answer", output: "boolean",
  judge: ({ result, row }) => ({ pass: result === row.output }),
});
const suite = defineEvalSuite({
  name: "Exact answers", dataset: selectedDatasetIdOrSlug, agent,
  evaluators: [{ evaluator: matchesReference, threshold: { equals: true } }],
});
```

Import both helpers from `raindrop-ai`. Use a comparison appropriate to the app;
this example assumes string output. The reference trace is available separately
when present. `requiresReference` means a trace requirement, not a plain-answer
requirement. Advanced portable evaluator downloads are unnecessary for this flow.

## Results

The receipt links each evaluator's saved run. MCP `get_eval_run` or SDK
`readEvalRun` can inspect the saved grades. One application run may contain several
evaluator runs. Record dataset/program versions, total graded rows, failures, and
execution errors. Give the user the saved command so rerunning does not depend on
chat history.
