---
name: vercel-ai-sdk
description: Vercel AI SDK reference documentation
---

# Vercel AI SDK Reference

The `@raindrop-ai/ai-sdk` package instruments the
[Vercel AI SDK](https://sdk.vercel.ai/) so wrapped `generateText`, `streamText`,
`generateObject`, and `streamObject` calls can be tracked in Raindrop.

## Installation

```bash
# npm
npm install @raindrop-ai/ai-sdk

# yarn
yarn add @raindrop-ai/ai-sdk

# pnpm
pnpm add @raindrop-ai/ai-sdk

# bun
bun add @raindrop-ai/ai-sdk
```

## Quick Start

```typescript
import * as ai from 'ai';
import { openai } from '@ai-sdk/openai';
import { createRaindropAISDK, eventMetadata } from '@raindrop-ai/ai-sdk';

// 1. Create the Raindrop client
const raindrop = createRaindropAISDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

// 2. Wrap the AI SDK with defaults
const { generateText, streamText } = raindrop.wrap(ai, {
  context: {
    userId, // REQUIRED: authenticated user id from your app
    eventName, // Optional: e.g. 'chat_message'
  },
  selfDiagnostics: { enabled: true },
});

// 3. Use the wrapped methods
const result = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Hello, world!',
  // Optional: override wrap defaults for this call
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({
      userId, // REQUIRED
      convoId, // Optional: conversation/thread id from your app
      eventName, // Optional override for this call
      properties: { source: requestSource },
    }),
  },
});

// Optional: identify user traits
await raindrop.users.identify({
  userId,
  traits: { plan: userPlan },
});

// 4. Flush before shutdown (serverless/scripts)
await raindrop.flush();
```

Wrapped `generateText`, `streamText`, `generateObject`, and `streamObject` calls
are now tracked.

## Runtime support

### Node.js

Use the default import:

```typescript
import { createRaindropAISDK } from '@raindrop-ai/ai-sdk';
```

### Cloudflare Workers

Cloudflare Workers supports a subset of Node's `AsyncLocalStorage` when you
enable `nodejs_compat`
([docs](https://developers.cloudflare.com/workers/runtime-apis/nodejs/asynclocalstorage/)).

Import the Workers entrypoint:

```typescript
import { createRaindropAISDK } from '@raindrop-ai/ai-sdk/workers';
```

If `nodejs_compat` is not enabled, `node:async_hooks` is not available and
AsyncLocalStorage-based context propagation cannot work.

### Edge / browser runtimes

Edge/browser runtimes don't provide Node `AsyncLocalStorage`. Event/traces
shipping still works, but context propagation across async boundaries is
disabled. If you need correlation across nested calls, pass an explicit
`eventId` with `eventMetadata()`.

---

## Configuration

### Client Options

```typescript
const raindrop = createRaindropAISDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!, // REQUIRED
  endpoint: 'https://api.raindrop.ai/v1', // Optional, defaults to production

  traces: {
    enabled: true, // Default: true
    flushIntervalMs: 1000, // Default: 1000
    maxBatchSize: 50, // Default: 50
    debug: false, // Logs trace shipping
    debugSpans: false, // Logs span parent relationships
  },

  events: {
    enabled: true, // Default: true
    partialFlushMs: 1000, // Debounce for streaming updates
    debug: false, // Logs event shipping
  },
});
```

### Wrap Options

```typescript
const { generateText } = raindrop.wrap(ai, {
  // Set defaults at wrap-time. Override per call with eventMetadata().
  context: {
    userId, // REQUIRED when setting user context here
    eventId, // Optional - use your own ID for correlation
    eventName, // Optional - categorize events
    convoId, // Optional - group events into conversations
    properties: {
      ...defaultProperties,
    }, // Optional - custom metadata
    attachments: [], // Optional - input/output attachments
  },
  selfDiagnostics: { enabled: true },

  autoAttachment: true, // Default: true. Set false to disable automatic parsing

  // Optional: customize event payload from messages
  buildEvent: (messages) => ({
    input: messages
      .filter((m) => m.role === 'user')
      .map((m) => m.content)
      .join('\n'),
    output: messages.filter((m) => m.role === 'assistant').pop()?.content,
    properties: { messageCount: messages.length },
  }),

  // Optional: control what gets sent
  send: {
    events: true, // Default: true
    traces: true, // Default: true
  },
});
```

---

## Context & Metadata

### Per-Call Context Override (Recommended)

Set stable defaults in `wrap()` and override them per call with
`eventMetadata()` when needed:

```typescript
import { createRaindropAISDK, eventMetadata } from '@raindrop-ai/ai-sdk';

// Wrap once with defaults
const { generateText } = raindrop.wrap(ai, {
  context: {
    userId,
    eventName,
    properties: {
      app: appName,
    },
  },
  selfDiagnostics: { enabled: true },
});

// Override defaults for this call
const result = await generateText({
  model: openai('gpt-4o'),
  prompt: 'Hello!',
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({
      userId, // REQUIRED per call when explicitly tracking
      convoId, // Conversation routing
      eventName, // Event categorization
      properties: { source: requestSource }, // Additional metadata
      // eventId, // Optional explicit eventId for correlation
    }),
  },
});
```

This keeps common values in one place while allowing each request to set or
override `userId`, `convoId`, `eventName`, `properties`, or `eventId`.

**Merge behavior:**

- Call-time values override wrap-time defaults
- `properties` are merged (call-time wins on conflicts)
- `eventId` is auto-generated if neither wrap-time nor call-time sets it

If you need full manual control over attachments, set `autoAttachment: false` in
`wrap()`.

`context` in `wrap()` is optional and useful for defaults. If both are
present, call-time `eventMetadata()` wins.

```typescript
const { generateText } = raindrop.wrap(ai, {
  context: {
    userId, // Fallback defaults
  },
  selfDiagnostics: { enabled: true },
});

await generateText({
  model,
  prompt,
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({ userId: actualUserId }),
  },
});
```

---

## Identifying Users

Use `users.identify` to associate traits with a user.

```typescript
await raindrop.users.identify({
  userId,
  traits: {
    email: userEmail,
    plan: userPlan,
    orgId,
  },
});
```

---

## Tools

Tools are automatically wrapped and traced. No additional configuration needed:

```typescript
import { tool } from 'ai';
import { z } from 'zod';
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const weatherTool = tool({
  description: 'Get current weather',
  parameters: z.object({ city: z.string() }),
  execute: async ({ city }) => {
    const data = await fetchWeather(city);
    return { temperature: data.temp, conditions: data.conditions };
  },
});

const { generateText } = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
});

// Tool calls are traced with args and results
const result = await generateText({
  model: openai('gpt-4o'),
  prompt: "What's the weather in Tokyo?",
  tools: { weather: weatherTool },
  maxSteps: 3,
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({
      userId,
      eventName,
    }),
  },
});
```

### Self Diagnostics

Enabling `selfDiagnostics` injects a hidden tool that the user's AI can call
silently when it hits an unrecoverable problem. Signals appear in Raindrop's
Self Diagnostics section and feed into issue discovery.

The three default categories are `missing_context`, `repeatedly_broken_tool`,
and `complete_task_failure`. You can replace these with your own.

```typescript
const { generateText } = raindrop.wrap(ai, {
  selfDiagnostics: {
    enabled: true,
    // Optional: replace defaults with domain-specific categories
    signals: {
      payment_failed: {
        description: 'Payment processing failed after retries.',
        sentiment: 'NEGATIVE',
      },
    },
    // Optional: extra guidance for the model
    guidance: "Only report when the user's transaction is actually blocked.",
  },
});
```

You don't need to mention self diagnostics in your system prompt - the SDK
handles the tool description automatically.

When the agent calls the tool, a signal is tracked on the same `eventId`:

```json
{
  "event_id": "evt_...",
  "signal_name": "self diagnostics - missing_context",
  "signal_type": "agent",
  "properties": {
    "source": "agent_reporting_tool",
    "category": "missing_context",
    "detail": "User asked to fix deployment but no access to logs or SSH credentials."
  }
}
```

### Nested LLM Calls in Tools

If your tool makes additional LLM calls, wrap them too for full trace
visibility:

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const summarizeTool = tool({
  description: 'Summarize text using AI',
  parameters: z.object({ text: z.string() }),
  execute: async ({ text }) => {
    // Inner wrapper - traces only, no separate event
    const { generateText: innerGenerate } = raindrop.wrap(ai, {
      selfDiagnostics: { enabled: true },
      send: { events: false, traces: true },
    });

    const summary = await innerGenerate({
      model: openai('gpt-4o-mini'),
      prompt: `Summarize: ${text}`,
      experimental_telemetry: {
        isEnabled: true,
        metadata: eventMetadata({ userId, eventId }), // Same eventId links traces
      },
    });

    return summary.text;
  },
});
```

---

## Streaming

Streaming methods (`streamText`, `streamObject`) work identically:

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const { streamText } = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
});

const result = await streamText({
  model: openai('gpt-4o'),
  prompt: 'Write a haiku about coding',
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({
      userId,
      eventName,
    }),
  },
});

// Consume the stream
for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

Raindrop captures streaming-specific metrics:

- Time to first chunk (`ai.stream.msToFirstChunk`)
- Total stream duration (`ai.stream.msToFinish`)
- Average output tokens per second (`ai.stream.avgOutputTokensPerSecond`)

### streamObject

`streamObject` works out of the box. You can await `result.object` directly
without manually consuming the stream:

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const { streamObject } = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
});

const result = await streamObject({
  model: openai('gpt-4o'),
  schema: z.object({ name: z.string(), age: z.number() }),
  prompt: 'Generate a person named Alice who is 30',
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({
      userId,
      eventName,
    }),
  },
});

// This works without consuming partialObjectStream
const person = await result.object;
```

---

## ToolLoopAgent

> Requires AI SDK v6 or later.

ToolLoopAgent is wrapped automatically. Both `generate()` and `stream()` trace tool
calls with their arguments and results. Pass per-call context via the top-level `metadata`
parameter.

### Typescript

`raindrop.wrap(ai)` changes the `ToolLoopAgent` method signatures to accept
top-level `metadata`, so typing against `ai.ToolLoopAgent` can cause TypeScript
errors. If you define your own agent type, wrap the AI SDK type with
`AgentWithMetadata` so `metadata` is still accepted on `generate()` and
`stream()`:

```typescript
import * as ai from 'ai';
import type { ToolSet } from 'ai';
import type { AgentWithMetadata } from '@raindrop-ai/ai-sdk';

export type YourToolLoopAgent = AgentWithMetadata<
  ai.ToolLoopAgent<unknown, ToolSet, any>
>;
```

### generate

```typescript
import * as ai from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';
import { createRaindropAISDK, eventMetadata } from '@raindrop-ai/ai-sdk';

const raindrop = createRaindropAISDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

const wrapped = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
});

const weatherTool = ai.tool({
  description: 'Get current weather',
  parameters: z.object({ city: z.string() }),
  execute: async ({ city }) => {
    return { temperature: 72, conditions: 'sunny' };
  },
});

const agent = new wrapped.ToolLoopAgent({
  model: openai('gpt-4o'),
  tools: { weather: weatherTool },
  maxSteps: 5,
});

const result = await agent.generate({
  prompt,
  metadata: eventMetadata({
    userId,
    eventName,
  }),
});
```

### stream

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const result = await agent.stream({
  prompt,
  metadata: eventMetadata({
    userId,
    eventName,
  }),
});

for await (const chunk of result.textStream) {
  process.stdout.write(chunk);
}
```

---

## Signals (Feedback)

Track user feedback on AI responses using the same `eventId`:

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const eventId = crypto.randomUUID();

const { generateText } = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
});

const result = await generateText({
  model: openai('gpt-4o'),
  prompt: userMessage,
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({
      userId,
      eventId,
      eventName,
    }),
  },
});

// Later, when user gives feedback
await raindrop.signals.track({
  eventId,
  name: 'thumbs_down',
  comment: 'Answer was incorrect',
});
```

### Signal Types

| Type         | Use Case                                       |
| ------------ | ---------------------------------------------- |
| `"default"`  | Generic signals (thumbs up/down)               |
| `"feedback"` | User comments about quality                    |
| `"edit"`     | User corrected the output                      |
| `"standard"` | Programmatic/custom instrumentation            |
| `"agent"`    | Self diagnostics issues from `selfDiagnostics` |

```typescript
// Thumbs up
await raindrop.signals.track({
  eventId,
  name: 'thumbs_up',
  sentiment: 'POSITIVE',
});

// User edit
await raindrop.signals.track({
  eventId,
  name: 'user_edit',
  type: 'edit',
  after: 'The corrected response text',
});
```

---

## Manual Event Updates

Update events after they're created:

```typescript
// Add properties
await raindrop.events.setProperties(eventId, {
  latencyMs: 1234,
  cached: true,
});

// Add attachments
await raindrop.events.addAttachments(eventId, [
  {
    type: 'code',
    name: 'generated.py',
    value: "print('hello')",
    role: 'output',
    language: 'python',
  },
]);

// Mark as complete with final output
await raindrop.events.finish(eventId, {
  output: 'Final response',
  model: 'gpt-4o',
});
```

---

## Custom Event Builder

Use `buildEvent` to customize the event payload from the conversation messages:

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const { generateText } = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
  buildEvent: (messages) => {
    const userMessages = messages.filter((m) => m.role === 'user');
    const assistantMessages = messages.filter((m) => m.role === 'assistant');

    return {
      input: userMessages.map((m) => m.content).join('\n'),
      output: assistantMessages.pop()?.content,
      eventName: customEventName,
      properties: {
        turnCount: userMessages.length,
        hasToolCalls: messages.some((m) => m.role === 'tool'),
      },
    };
  },
});

await generateText({
  model: openai('gpt-4o'),
  prompt: 'Summarize this conversation',
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({ userId, eventName }),
  },
});
```

---

## Flush & Shutdown

Always flush before your process exits to ensure all data is sent:

```typescript
// Serverless: flush at end of request
export async function POST(request: Request) {
  const result = await generateText({ ... });
  await raindrop.flush();
  return Response.json({ text: result.text });
}

// Long-running: shutdown gracefully
process.on("SIGTERM", async () => {
  await raindrop.shutdown();
  process.exit(0);
});
```

---

## Debugging

Enable debug logging to troubleshoot issues:

```typescript
const raindrop = createRaindropAISDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  events: { debug: true },
  traces: { debug: true, debugSpans: true },
});
```

This logs:

- Every event sent to Raindrop
- Every trace batch shipped
- Span parent/child relationships (with `debugSpans`)

---

## Advanced: Context Propagation APIs

For advanced use cases, access the underlying context propagation utilities:

> These APIs use AsyncLocalStorage and are only effective in Node.js (and
> Cloudflare Workers with `nodejs_compat`). In Edge/browser runtimes they behave
> like no-ops.

```typescript
import {
  currentSpan,
  withCurrent,
  getContextManager,
} from '@raindrop-ai/ai-sdk';

// Get the current span (returns NOOP_SPAN if none)
const span = currentSpan();
console.log(span.eventId, span.traceIdB64);

// Run code within a span context
withCurrent(span, () => {
  // Nested calls inherit this span as parent
  const nested = currentSpan();
  console.log(nested.spanIdB64 === span.spanIdB64); // true
});

// Access the context manager directly
const cm = getContextManager();
const parentIds = cm.getParentSpanIds();
```

---

## Troubleshooting

### Events not appearing in dashboard

1. **Check your write key** - Ensure `RAINDROP_WRITE_KEY` is set correctly
2. **Flush before exit** - Call `await raindrop.flush()` before your process
   ends
3. **Enable debug logging** - Set `events: { debug: true }` to see what's being
   sent

### Traces missing or incomplete

1. **Enable trace debugging** - Set `traces: { debug: true, debugSpans: true }`
2. **Check for errors** - Look for `[raindrop-ai/ai-sdk]` prefixed logs

### Context not propagating to nested calls

1. **Check your runtime** - AsyncLocalStorage-based propagation requires Node.js
   (or Cloudflare Workers with `nodejs_compat`)
2. **Workers import** - On Cloudflare Workers, import
   `@raindrop-ai/ai-sdk/workers`

Ensure nested AI calls use the same `eventId` and set `send: { events: false }`
to avoid duplicate events:

```typescript
import { eventMetadata } from '@raindrop-ai/ai-sdk';

const { generateText: innerGenerate } = raindrop.wrap(ai, {
  selfDiagnostics: { enabled: true },
  send: { events: false, traces: true },
});

await innerGenerate({
  model,
  prompt,
  experimental_telemetry: {
    isEnabled: true,
    metadata: eventMetadata({ userId, eventId }),
  },
});
```

---

Use the Raindrop dashboard to verify events and traces after integration.
