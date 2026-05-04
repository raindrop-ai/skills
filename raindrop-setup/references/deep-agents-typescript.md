---
name: deep-agents-typescript
description: Deep Agents (TypeScript) integration reference
---

# Deep Agents (TypeScript)

The `@raindrop-ai/deep-agents` package attaches a Raindrop callback handler to LangChain Deep Agents so every planner, executor, and tool step is captured.

**Detect:** imports from `deepagents` (e.g. `createDeepAgent`).
**Docs:** https://raindrop.ai/docs/integrations/deepagents

## Installation

```bash
# npm
npm install @raindrop-ai/deep-agents

# pnpm
pnpm add @raindrop-ai/deep-agents

# yarn
yarn add @raindrop-ai/deep-agents

# bun
bun add @raindrop-ai/deep-agents
```

## Quick Start

```typescript
import { createDeepAgent } from "deepagents";
import { ChatAnthropic } from "@langchain/anthropic";
import { createRaindropDeepAgents } from "@raindrop-ai/deep-agents";

const raindrop = createRaindropDeepAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const agent = createDeepAgent({
  model: new ChatAnthropic({ model: "claude-sonnet-4-20250514" }),
  tools: [/* filesystem, shell, etc. */],
});

const result = await agent.invoke(
  { messages: [{ role: "user", content: "Hello, world!" }] },
  { callbacks: [raindrop.handler] },
);

// ...

await raindrop.shutdown(); // publish queued events before shutdown
```

## Notes

- Pass `raindrop.handler` via `{ callbacks: [...] }` on every `invoke` / `stream` call you want tracked.
