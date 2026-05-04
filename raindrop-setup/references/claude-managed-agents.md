---
name: claude-managed-agents
description: Claude Managed Agents (TypeScript) integration reference
---

# Claude Managed Agents (TypeScript)

The `@raindrop-ai/claude-managed-agents` package wraps the Anthropic SDK so every Anthropic-hosted managed agent session, message, and tool call is captured.

**Detect:** imports from `@anthropic-ai/sdk` *and* use of `client.beta.sessions` / `client.beta.agents` / `client.beta.environments` (Claude's managed agent runtime). If only `client.messages.create` is used, this isn't the right integration.
**Docs:** https://raindrop.ai/docs/integrations/claude-managed-agents

## Installation

```bash
# npm
npm install @raindrop-ai/claude-managed-agents

# pnpm
pnpm add @raindrop-ai/claude-managed-agents

# yarn
yarn add @raindrop-ai/claude-managed-agents

# bun
bun add @raindrop-ai/claude-managed-agents
```

## Quick Start

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { createRaindropClaudeManagedAgents } from "@raindrop-ai/claude-managed-agents";

const raindrop = createRaindropClaudeManagedAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

const client = new Anthropic();
const wrapped = raindrop.wrap(client, {
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const session = await wrapped.beta.sessions.create({
  agent: agentId,
  environment_id: envId,
  title: "Hello, world!",
});

// ...

await raindrop.shutdown(); // publish queued events before shutdown
```

## Notes

- Wrap the Anthropic client *per conversation* — `userId` and `convoId` go on the wrap call, not the constructor.
