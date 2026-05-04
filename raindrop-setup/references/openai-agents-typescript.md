---
name: openai-agents-typescript
description: OpenAI Agents SDK (TypeScript) integration reference
---

# OpenAI Agents SDK (TypeScript)

The `@raindrop-ai/openai-agents` package registers a Raindrop trace processor with the OpenAI Agents runtime so every run, handoff, and tool call is captured.

**Detect:** imports from `@openai/agents` (e.g. `Agent`, `run`, `addTraceProcessor`).
**Docs:** https://raindrop.ai/docs/integrations/openai-agents

## Installation

```bash
# npm
npm install @raindrop-ai/openai-agents

# pnpm
pnpm add @raindrop-ai/openai-agents

# yarn
yarn add @raindrop-ai/openai-agents

# bun
bun add @raindrop-ai/openai-agents
```

## Quick Start

```typescript
import { createRaindropOpenAIAgents } from "@raindrop-ai/openai-agents";
import { Agent, run, addTraceProcessor } from "@openai/agents";

const raindrop = createRaindropOpenAIAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

addTraceProcessor(raindrop.processor);

const agent = new Agent({
  name: "Assistant",
  model: "gpt-4o",
  instructions: "Be helpful",
});

const result = await run(agent, "Hello, world!");

// ...

await raindrop.flush(); // publish queued events before shutdown
```

## Notes

- Call `addTraceProcessor` once at boot — every subsequent agent run gets traced automatically.
