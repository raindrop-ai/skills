# Evaluator contract

Hosted programs run once per trace. Return one `{ pass, note? }`, `{ score, note? }`
(integer 1–5), or `{ value, note? }` (finite number), matching the evaluator's
output type. A failing verdict is a successful execution. Throw on execution
failure; orchestration records `errored` and continues other traces. Missing or
invalid returns are errors. Historical `ungraded` records remain readable.

TypeScript context fields are `trace`, `expectedOutput`, `referenceTrace`, and
`rowMetadata`. Python local callbacks expose `trace`, `expected_output`,
`reference_trace`, and `row_metadata`. Optional expectations and references can
be absent; row metadata defaults to an empty object.

- `trace`: the candidate's canonical `{ origin, event, entries, caseContext? }`.
- `entries`: ordered rich interactions, using the same discriminated schema as
  the trace viewer. Narrow by `entry.type` before reading variant fields.
- `expectedOutput`: the dataset row's authored output, including structured JSON.
- `rowMetadata`: the dataset row's custom properties, such as `toneType`.
- `referenceTrace`: the captured baseline, when available.

Keep dataset metadata separate from `trace.event.properties` and
`referenceTrace.event.properties`. Neither event's metadata overrides row columns.
A reference trace is evidence of prior behavior, not automatically a correct answer.

Hosted `run_judge(text, rubric)` returns `{ pass, note }`.
`run_judge(text, rubric, { output: "score" })` returns `{ score, note }`.
Both accept explicit free-text evidence and throw on failure. No metadata or
reference is silently appended. Judge calls may be intermediate steps and their
output type is independent of the final verdict. Each evaluator checks one criterion.
`formatTrace(trace)` from `./stdlib` serializes the full canonical trace without
additional clipping; upstream truncation markers remain visible.

Hosted invocations run in separate workers (20 at a time), with a 60-second
per-trace deadline. Provider calls queue at concurrency 20 and have a 30-second
deadline including up to two retries. There is no separate judge-call-count cap.
Portable runtimes default to four concurrent provider callbacks, 30 seconds per
callback and 60 seconds for a judge program; limits are configurable. Portable
sandbox resource failures reject execution; the calling runner must handle them.

Local evaluators use the customer's model libraries; hosted `run_judge` is not
injected into local callbacks. Existing local `row`, `result`, and `reference`
fields remain available: `result` is the original application return object, and
`reference` retains both its snapshot and trace. The lower-level TypeScript replay
API also retains `case`. Existing verdict shapes and Python verdict models remain
valid. No output wrapper or contractVersion is required.

New portable exports use runtime ABI `raindrop-eval-v3` and scope `trace`.
After upgrading an SDK, re-pull saved portable artifacts; legacy batch artifacts
are rejected rather than silently executed with changed semantics. Ordinary local
callbacks and their existing submission protocol require no rewrite.
