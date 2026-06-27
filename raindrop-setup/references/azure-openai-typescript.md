---
name: azure-openai-typescript
description: Azure OpenAI (TypeScript) integration reference
---

# Azure OpenAI (TypeScript)

The `@raindrop-ai/azure-openai` package wraps the `AzureOpenAI` client so every chat completion is captured.

**Detect:** `AzureOpenAI` imported from `openai`, or `AZURE_OPENAI_*` environment variables in use.
**Docs:** https://raindrop.ai/docs/integrations/azure-openai

## Installation

```bash
# npm
npm install @raindrop-ai/azure-openai

# pnpm
pnpm add @raindrop-ai/azure-openai

# yarn
yarn add @raindrop-ai/azure-openai

# bun
bun add @raindrop-ai/azure-openai
```

## Quick Start

```typescript
import { createRaindropAzureOpenAI } from "@raindrop-ai/azure-openai";
import { AzureOpenAI } from "openai";

const raindrop = createRaindropAzureOpenAI({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const client = new AzureOpenAI({
  endpoint: process.env.AZURE_OPENAI_ENDPOINT!,
  apiKey: process.env.AZURE_OPENAI_KEY!,
  apiVersion: "2024-10-21",
});

const wrapped = raindrop.wrap(client);

const result = await wrapped.chat.completions.create({
  model: "gpt-4o-mini",
  messages: [{ role: "user", content: "Hello, world!" }],
});

// ...

await raindrop.shutdown(); // publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropAzureOpenAI({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.
