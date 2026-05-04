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
