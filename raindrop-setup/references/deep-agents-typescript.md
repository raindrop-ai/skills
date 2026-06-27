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

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropDeepAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Pass `raindrop.handler` via `{ callbacks: [...] }` on every `invoke` / `stream` call you want tracked.
