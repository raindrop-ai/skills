---
name: ash
description: Ash (TypeScript) integration reference
---

# Ash (TypeScript)

The `@raindrop-ai/ash` package plugs into Ash's native `agent/instrumentation.ts` entry point so every agent turn, sub-agent dispatch (across V8 sandboxes), tool call, and model call ships to Raindrop + Workshop. One file, no per-call wrapping.

**Detect:** imports from `experimental-ash` / `experimental-ash/context`, or an existing `agent/instrumentation.ts` in the project root.
**Docs:** https://raindrop.ai/docs/integrations/ash

## Installation

```bash
# npm
npm install @raindrop-ai/ash

# pnpm
pnpm add @raindrop-ai/ash

# yarn
yarn add @raindrop-ai/ash

# bun
bun add @raindrop-ai/ash
```

`@raindrop-ai/ash` peer-depends on `@vercel/otel`, `@opentelemetry/api`, `@opentelemetry/sdk-trace-base`, `experimental-ash@>=0.23.0`, and `ai@>=7.0.0-canary.0`. In a real Ash project these are already installed.

## Quick Start

Drop a single file at `agent/instrumentation.ts` (Ash auto-discovers it at server startup):

```typescript
// agent/instrumentation.ts
import { registerOTel } from "@vercel/otel";
import { defineRaindropInstrumentation } from "@raindrop-ai/ash";

export default defineRaindropInstrumentation({
  registerOTel,
  writeKey: process.env.RAINDROP_WRITE_KEY,
  metadata: {
    "raindrop.userId": "user_123", // id for end user
    "raindrop.convoId": "convo_456", // id for conversation
  },
});
```

That's the whole integration. No `wrap()`, no `getTracer()`, no per-call setup.

## Notes

- **Sub-agents across sandboxes.** Each Ash sub-agent runs in its own V8 sandbox. The integration uses Ash's first-class `getSession()` API (added in `experimental-ash@0.23.0`) to detect when the current sandbox is a sub-agent and lifts the parent's identity onto the event metadata (`raindrop.parent.{sessionId,turnId,turnSequence}`, `raindrop.subagent.name`, `raindrop.agent.role`). Workshop renders sub-agents as nested AGENT blocks; the dashboard stitches sub-agent events under the parent's turn.
- **Workshop mirroring.** During `ash dev`, Raindrop also ships to the local Workshop daemon if `RAINDROP_WORKSHOP=1` or a Workshop write key is set. The same events go to production when a real `RAINDROP_WRITE_KEY` is configured.
- **No client to shutdown.** Ash owns the OTel lifecycle. The integration's exporter drains on `SIGINT` / `SIGTERM` via the shutdown hook Ash already calls.
- **Don't pre-wrap.** If the project already has `agent/instrumentation.ts`, edit it in place — do not introduce a second instrumentation file or wrap calls manually. Ash only loads one `agent/instrumentation.ts`.
- **Identifying users.** Pass `raindrop.userId` / `raindrop.convoId` via `metadata` (top-level keys are merged onto every event) or set them per-turn via Ash's `runtimeContext` if the values are dynamic. The integration also accepts `getMetadata?: (event) => Record<string, string>` for fully dynamic enrichment.
