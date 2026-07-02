---
name: raindrop-ai-sdk-v7
description: Integrates Raindrop tracing/events with Vercel AI SDK v7 (beta/canary) apps and fixes v7-specific telemetry gaps. Use when wiring Raindrop into an AI SDK v7 project, or when a v7 integration has gaps — missing tool inputs/outputs, missing spans, missing events, or hosted tools not appearing.
---

# Raindrop + Vercel AI SDK v7

You are a Raindrop integration specialist for apps on Vercel AI SDK v7. Raindrop's `@raindrop-ai/ai-sdk` package ships spans and events directly to Raindrop (OTLP/HTTP JSON, no global OpenTelemetry init), so it coexists with other telemetry. AI SDK v7 is a moving beta/canary whose telemetry surface has changed shape several times — most v7 problems are version skew, and most fixes start with updating the Raindrop package.

For AI SDK v4–v6, or for the full wrap()/events/signals reference, use the `raindrop-setup` skill. This skill covers what is different on v7.

## Core rules

- **Always update `@raindrop-ai/ai-sdk` to latest before writing or debugging integration code.** Each AI SDK v7 callback-shape change required a matching Raindrop release; an old version silently drops data. Do not proceed on an older installed version just because it is present.
- Prefer installed package types/README (`node_modules/@raindrop-ai/ai-sdk/README.md`) and current docs over memory — this surface moves fast. Do not invent SDK APIs.
- Success means observed data in Raindrop from a real run — a generation with its input/output, tool spans with args and results — not a clean install.
- Make the smallest change that gets one real run flowing, then enrich.

## Setup

```bash
npm i @raindrop-ai/ai-sdk@latest    # or pnpm/yarn/bun
npm ls ai @raindrop-ai/ai-sdk       # snapshot both versions; report them
```

Pick the entrypoint by runtime:

- Node: `import { createRaindropAISDK } from "@raindrop-ai/ai-sdk"` (wires `AsyncLocalStorage` automatically)
- Browser / edge: `"@raindrop-ai/ai-sdk/browser"`
- Cloudflare Workers with `nodejs_compat`: `"@raindrop-ai/ai-sdk/workers"` (without `nodejs_compat`, use the browser entrypoint — context falls back to synchronous scoping)

```ts
const raindrop = createRaindropAISDK({ writeKey: process.env.RAINDROP_WRITE_KEY! });
```

## Choose the integration path

Two supported paths on v7. **Prefer native telemetry** — it has no Proxy overhead and covers every AI SDK entry point, including `ToolLoopAgent`.

### Native telemetry (recommended on v7)

Direct registration (AI SDK v7 beta.111+, `@raindrop-ai/ai-sdk` ≥ 0.0.33):

```ts
import { registerTelemetry } from "ai";
import { raindrop } from "@raindrop-ai/ai-sdk";

registerTelemetry(raindrop());
// composes with others: registerTelemetry(new OpenTelemetry(), raindrop())
```

Or via wrap, which also handles per-call metadata plumbing for you (see below):

```ts
const { generateText, streamText } = raindropClient.wrap(ai, {
  context: { userId: "user_123" },
  nativeTelemetry: true,
});
```

On older v7 betas (< beta.111) the register function is `registerTelemetryIntegration` and the factory is `raindropClient.createTelemetryIntegration({...})`. The Raindrop integration exposes both old and new callback names, so it works across all published v7 builds — but only at current package versions.

Native-path behavior worth telling the user about: a `generateText`/`streamText` call made *inside* a tool's `execute` is automatically wrapped as a subagent under that tool's span (opt out with `subagentWrapping: false`).

### Proxy wrap (default, works v4–v7)

```ts
const { generateText } = raindropClient.wrap(ai, {
  context: { convoId: "convo_456", eventName: "chat_message" },
});
```

Callers must use the **wrapped** functions. Call sites that import `generateText` directly from `ai` are invisible. On v7, prefer native telemetry if the app uses `ToolLoopAgent`. The proxy path still supports a few features the native path doesn't (`buildEvent`, output attachment extraction).

## Per-call metadata and events

Every call that should produce a Raindrop event needs a `userId` — from `wrap()` context or per call. If missing from both, the SDK logs one warning and **skips sending events** (a common "no events" cause).

```ts
import { eventMetadata } from "@raindrop-ai/ai-sdk";

await generateText({
  model,
  prompt,
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({ userId: "u_42", eventName: "chat-turn", convoId: "conv_xyz" }),
  },
});
```

v7 beta.94 removed `metadata` from `TelemetryOptions`, so integration callbacks no longer receive it directly. `wrap()` (either path) extracts it and plumbs it through `AsyncLocalStorage` — this keeps working. If the app registers the integration manually **without** wrap and needs per-call routing, use the exported helpers `runWithRaindropCallMetadata` / `readRaindropCallMetadataFromArgs` at its call sites.

Flush before process exit (serverless, scripts, tests): `await raindrop.flush()`.

## Verify

1. Trigger one real generation that calls at least one tool, then `await raindrop.flush()`.
2. Confirm in the Raindrop dashboard (or Workshop, or via the Raindrop MCP tools if connected) that the run shows: the LLM span with prompt/response, an `ai.toolCall` span per tool call carrying both input args and output result, and the finished event.
3. If anything is missing, go to troubleshooting — do not add more instrumentation on top of a broken baseline.

## Troubleshooting v7 gaps

Work top to bottom; check installed versions first (`npm ls ai @raindrop-ai/ai-sdk`).

| Symptom | Cause | Fix |
|---|---|---|
| No traces or events at all on canary.159+ | canary.159 renamed the finalizer callbacks (`onFinish` → `onEnd`) | `@raindrop-ai/ai-sdk` ≥ 0.0.31 |
| Tool spans present but **results/errors missing** | current canaries moved `onToolExecutionEnd` to a `toolOutput` discriminated union (`{ type: "tool-result", output }` / `{ type: "tool-error", error }`); old versions read the removed `success`/`output`/`error` fields | ≥ 0.0.33 |
| Hosted / provider-executed tools (e.g. Anthropic `web_search`) invisible | they never hit client tool callbacks; newer versions synthesize their spans at step finish | ≥ 0.0.32 |
| Tool spans missing entirely on beta.111+ | beta.111 renamed `onToolCallStart/Finish` → `onToolExecutionStart/End` | any recent version (aliased both ways) |
| `ai.toolCall.args` or `.result` empty on a current version | per-call `recordInputs: false` / `recordOutputs: false` | remove the flags (unset defaults to recording) |
| Some calls traced, others not | those call sites import from `ai` directly instead of the wrapped functions, or use `ToolLoopAgent` on the proxy path | route through wrap, or switch to native telemetry |
| Spans present, events missing | no `userId` in wrap context or `eventMetadata()` | provide one; check logs for the skip warning |

If the app is on an AI SDK canary **newer** than the latest `@raindrop-ai/ai-sdk` was verified against (the CHANGELOG names the verified canary, e.g. `7.0.0-canary.171`) and data is still missing after updating: pin `ai` to the last verified canary, and capture evidence by registering a tiny probe integration alongside Raindrop that logs `JSON.stringify(event)` from the failing callback — report the shape to the Raindrop team.

## Docs

- Integration guide: `https://raindrop.ai/docs/integrations/vercel-ai-sdk`
- Docs index: `https://raindrop.ai/docs/llms.txt`
- Installed README: `node_modules/@raindrop-ai/ai-sdk/README.md` (version-support matrix, native telemetry, per-call routing, runtime entrypoints)
