---
name: strands-typescript
description: Strands Agents (TypeScript) integration reference
---

# Strands Agents (TypeScript)

The `@raindrop-ai/strands` package registers hooks on a Strands `Agent` so every run and tool call is captured.

**Detect:** imports from `@strands-agents/sdk` (e.g. `Agent`).
**Docs:** https://raindrop.ai/docs/integrations/strands

## Installation

```bash
# npm
npm install @raindrop-ai/strands

# pnpm
pnpm add @raindrop-ai/strands

# yarn
yarn add @raindrop-ai/strands

# bun
bun add @raindrop-ai/strands
```

## Quick Start

```typescript
import { Agent } from "@strands-agents/sdk";
import { createRaindropStrandsAgents } from "@raindrop-ai/strands";

const raindrop = createRaindropStrandsAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const agent = new Agent({
  model: "us.amazon.nova-lite-v1:0",
  systemPrompt: "Be helpful",
});

raindrop.handler.registerHooks(agent);

const result = await agent("Hello, world!");

// ...

await raindrop.flush(); // publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropStrandsAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- `registerHooks` mutates the agent in place — call it once after creation.
