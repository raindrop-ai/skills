---
name: temporal
description: Temporal (TypeScript) integration reference
---

# Temporal (TypeScript)

The `@raindrop-ai/temporal` package wires interceptors and sinks into a Temporal `Worker` so workflows and activities stitch into a single Raindrop trace.

**Detect:** imports from `@temporalio/worker`, `@temporalio/workflow`, `@temporalio/activity`, or `@temporalio/client`.
**Docs:** https://raindrop.ai/docs/integrations/temporal

## Installation

```bash
# npm
npm install @raindrop-ai/temporal

# pnpm
pnpm add @raindrop-ai/temporal

# yarn
yarn add @raindrop-ai/temporal

# bun
bun add @raindrop-ai/temporal
```

## Quick Start

```typescript
import { Worker } from "@temporalio/worker";
import { createRaindropTemporal } from "@raindrop-ai/temporal";
import * as activities from "./activities";

const raindrop = createRaindropTemporal({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const worker = await Worker.create({
  taskQueue: "my-queue",
  workflowsPath: require.resolve("./workflows"),
  activities,
  interceptors: {
    activity: [raindrop.interceptors.activityFactory],
    workflowModules: ["@raindrop-ai/temporal/workflow-interceptors"],
  },
  sinks: raindrop.sinks,
});

process.on("SIGINT", async () => {
  await worker.shutdown();
  await raindrop.shutdown(); // publish queued events before shutdown
});

await worker.run();
```

## Notes

- Both interceptors and sinks must be wired — workflows use the workflow-side interceptor module path; activities use the factory.
- Shutdown the Raindrop client after the Worker shuts down so trailing events flush.
