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

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropClaudeManagedAgents({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Wrap the Anthropic client *per conversation* — `userId` and `convoId` go on the wrap call, not the constructor.
