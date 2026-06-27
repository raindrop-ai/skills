---
name: bedrock-typescript
description: AWS Bedrock (TypeScript) integration reference
---

# AWS Bedrock (TypeScript)

The `@raindrop-ai/bedrock` package wraps `BedrockRuntimeClient` so every `Converse` and `InvokeModel` call is captured.

**Detect:** imports from `@aws-sdk/client-bedrock-runtime` (e.g. `BedrockRuntimeClient`, `ConverseCommand`, `InvokeModelCommand`).
**Docs:** https://raindrop.ai/docs/integrations/bedrock

## Installation

```bash
# npm
npm install @raindrop-ai/bedrock

# pnpm
pnpm add @raindrop-ai/bedrock

# yarn
yarn add @raindrop-ai/bedrock

# bun
bun add @raindrop-ai/bedrock
```

## Quick Start

```typescript
import { createRaindropBedrock } from "@raindrop-ai/bedrock";
import {
  BedrockRuntimeClient,
  ConverseCommand,
} from "@aws-sdk/client-bedrock-runtime";

const raindrop = createRaindropBedrock({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const client = new BedrockRuntimeClient({ region: "us-east-1" });
const wrapped = raindrop.wrap(client);

const result = await wrapped.send(
  new ConverseCommand({
    modelId: "anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages: [{ role: "user", content: [{ text: "Hello, world!" }] }],
  }),
);

// ...

await raindrop.flush(); // publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropBedrock({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Use the wrapped client for all subsequent `.send(...)` calls — the unwrapped one is unobserved.
