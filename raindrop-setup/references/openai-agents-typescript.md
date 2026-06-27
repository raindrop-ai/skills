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

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropOpenAIAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Call `addTraceProcessor` once at boot — every subsequent agent run gets traced automatically.
