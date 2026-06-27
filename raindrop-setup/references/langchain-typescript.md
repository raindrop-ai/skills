---
name: langchain-typescript
description: LangChain (TypeScript) integration reference
---

# LangChain (TypeScript)

The `@raindrop-ai/langchain` package attaches a Raindrop callback handler to LangChain and LangGraph runs so every chain, model, and tool step is captured.

**Detect:** imports from `@langchain/core`, `@langchain/openai`, `@langchain/anthropic`, `langchain`, or `@langchain/langgraph`.
**Docs:** https://raindrop.ai/docs/integrations/langchain

## Installation

```bash
# npm
npm install @raindrop-ai/langchain

# pnpm
pnpm add @raindrop-ai/langchain

# yarn
yarn add @raindrop-ai/langchain

# bun
bun add @raindrop-ai/langchain
```

## Quick Start

```typescript
import { createRaindropLangChain } from "@raindrop-ai/langchain";
import { ChatOpenAI } from "@langchain/openai";
import { HumanMessage } from "@langchain/core/messages";

const raindrop = createRaindropLangChain({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const model = new ChatOpenAI({ model: "gpt-4o" });

const result = await model.invoke(
  [new HumanMessage("Hello, world!")],
  { callbacks: [raindrop.handler] },
);

// ...

await raindrop.flush(); // publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropLangChain({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Pass `raindrop.handler` via `{ callbacks: [...] }` on every invoke / stream call you want tracked.
- For long-running processes, `flush()` before shutdown so queued events publish.
