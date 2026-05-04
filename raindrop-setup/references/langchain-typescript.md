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

## Notes

- Pass `raindrop.handler` via `{ callbacks: [...] }` on every invoke / stream call you want tracked.
- For long-running processes, `flush()` before shutdown so queued events publish.
