---
name: mastra
description: Mastra (TypeScript) integration reference
---

# Mastra (TypeScript)

The `@raindrop-ai/mastra` package wraps a Mastra `Agent` so every `generate` and `stream` call is captured.

**Detect:** imports from `@mastra/core` (e.g. `@mastra/core/agent`).
**Docs:** https://raindrop.ai/docs/integrations/mastra

## Installation

```bash
# npm
npm install @raindrop-ai/mastra

# pnpm
pnpm add @raindrop-ai/mastra

# yarn
yarn add @raindrop-ai/mastra

# bun
bun add @raindrop-ai/mastra
```

## Quick Start

```typescript
import { createRaindropMastra } from "@raindrop-ai/mastra";
import { Agent } from "@mastra/core/agent";
import { openai } from "@ai-sdk/openai";

const raindrop = createRaindropMastra({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const agent = new Agent({
  name: "Assistant",
  model: openai("gpt-4o-mini"),
  instructions: "Be helpful",
});

const wrapped = raindrop.wrap(agent);

const result = await wrapped.generate("Hello, world!");

// ...

await raindrop.shutdown(); // publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropMastra({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.
