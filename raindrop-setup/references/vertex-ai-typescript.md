---
name: vertex-ai-typescript
description: Vertex AI / Google Gen AI (TypeScript) integration reference
---

# Vertex AI (TypeScript)

The `@raindrop-ai/vertex-ai` package wraps the `GoogleGenAI` client so every `generateContent` call is captured.

**Detect:** imports from `@google/genai` (e.g. `GoogleGenAI`).
**Docs:** https://raindrop.ai/docs/integrations/vertex-ai

## Installation

```bash
# npm
npm install @raindrop-ai/vertex-ai

# pnpm
pnpm add @raindrop-ai/vertex-ai

# yarn
yarn add @raindrop-ai/vertex-ai

# bun
bun add @raindrop-ai/vertex-ai
```

## Quick Start

```typescript
import { createRaindropVertexAI } from "@raindrop-ai/vertex-ai";
import { GoogleGenAI } from "@google/genai";

const raindrop = createRaindropVertexAI({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  userId: "user_123", // id for end user
  convoId: "convo_456", // id for conversation
});

const client = new GoogleGenAI({ apiKey: process.env.GOOGLE_API_KEY! });
const wrapped = raindrop.wrap(client);

const result = await wrapped.models.generateContent({
  model: "gemini-2.0-flash",
  contents: "Hello, world!",
});

// ...

await raindrop.shutdown(); // publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `projectId`:

```typescript
const raindrop = createRaindropVertexAI({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  projectId: "support-prod",
});
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `projectId` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Use the wrapped client for all subsequent calls.
- Shutdown uses `raindrop.shutdown()` (not `flush()`).
