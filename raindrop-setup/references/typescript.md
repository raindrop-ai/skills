---
name: typescript
description: TypeScript SDK reference documentation
---

# TypeScript SDK Reference

## Installation

Install using the project's package manager:

```bash
# npm
npm install raindrop-ai

# yarn
yarn add raindrop-ai

# bun
bun add raindrop-ai
```

```typescript
import { Raindrop } from 'raindrop-ai';

// RAINDROP_WRITE_KEY should have already been loaded from environment variables
const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});
```

---

## Quick Start: Interaction API

The Interaction API uses a simple three-step pattern:

1. **`begin()`** - Start interaction; pass AI data (`model`, `input`, `convoId`)
   here.
2. **Update** - Optional: `setProperty` / `setProperties` / `addAttachments` for
   **custom metadata only** (not `model`/`input`/`output`/`convoId`).
3. **`finish()`** - End interaction; pass `output` and optionally `model`.

For event tracking calls (`begin()` and `trackAi()`), both `event` and `userId`
are **REQUIRED**.

### Example: Chat Completion

```typescript
import { openai } from '@ai-sdk/openai';
import { generateText } from 'ai';
// UUID generation: check the project's existing approach first.
// - If the project already uses a UUID library (e.g. 'uuid'), use that.
// - If using Next.js with Turbopack, use crypto.randomUUID() or the 'uuid' package — Turbopack does not polyfill Node built-ins.
// - Otherwise, `import { randomUUID } from 'crypto'` is fine for Node.js servers (webpack, esbuild, etc. handle it).
import { randomUUID } from 'crypto';
import { Raindrop } from 'raindrop-ai';

const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

const eventId = randomUUID(); // Only generate if the app doesn't already have an event ID

// 1. Start the interaction — use actual values from your app
const interaction = raindrop.begin({
  eventId,
  event: eventName, // REQUIRED: descriptive name for this AI action (e.g. 'chat_message', 'code_generation')
  userId: userId, // REQUIRED: authenticated user's unique identifier from your app
  input: userMessage, // The actual user message/prompt
  model: modelName, // The model being called (e.g. 'gpt-4o')
  convoId: conversationId, // Your app's conversation/thread ID (if applicable)
  properties: {
    // Optional: any custom metadata relevant to your app
    system_prompt: systemPrompt,
  },
});

// 2. Make the LLM call
const { text } = await generateText({
  model: openai('gpt-4o'),
  prompt: userMessage,
});

// 3. Finish the interaction
interaction.finish({
  output: text,
});
```

### Updating an Interaction

Use `setProperty`, `setProperties`, or `addAttachments` for custom metadata and
attachments. Do not use them for `model`, `input`, `output`, or `convoId` (use
`begin()` / `finish()` instead; see
[AI data vs properties](#ai-data-fields-vs-custom-properties)).

```typescript
interaction.setProperty('stage', 'embedding');
interaction.addAttachments([
  {
    type: 'text',
    name: 'Additional Info',
    value: 'A very long document',
    role: 'input',
  },
  { type: 'image', value: 'https://example.com/image.png', role: 'output' },
  {
    type: 'iframe',
    name: 'Generated UI',
    value: 'https://newui.generated.com',
    role: 'output',
  },
]);
```

### AI data fields vs custom properties

**Rule:** `model`, `input`, `output`, `convoId` are **AI data** - set only via
`begin()`, `finish()`, or `trackAi()`. `setProperty()` / `setProperties()` only
write to **custom properties**; they do not update AI data or UI badges.

| Field                                 | Set via                                    | Not set via     |
| ------------------------------------- | ------------------------------------------ | --------------- |
| `model`, `input`, `output`, `convoId` | `begin()`, `finish()`, `trackAi()`         | `setProperty()` |
| Other metadata                        | `setProperty()`, `properties` in `begin()` | -               |

- **Wrong:** `interaction.setProperty('model', 'gpt-4o')` -> adds a property
  only; Model badge stays empty.
- **Correct:** pass `model` in `begin({ ... model: 'gpt-4o' })` or
  `finish({ output: '...', model: 'gpt-4o' })`.

### Resuming an Interaction

To resume an interaction without the original object returned from `begin()`,
use `resumeInteraction()`:

```typescript
const interaction = raindrop.resumeInteraction(eventId);
```

---

## Single-Shot Tracking (`trackAi`)

For simple request-response interactions, use `trackAi()`; pass all AI data
(`model`, `input`, `output`, `convoId`) in that call (not via `setProperty()`).

Note: `input` and `output` should be the actual prompt and response from your
model, not placeholder strings.

```typescript
raindrop.trackAi({
  event: eventName, // REQUIRED: descriptive name for this AI action (e.g. 'chat_message', 'code_generation')
  userId: userId, // REQUIRED: authenticated user's unique identifier from your app
  model: modelName, // The model being called (e.g. 'gpt-4o-mini')
  input: userPrompt, // The actual prompt sent to the model
  output: modelResponse, // The actual response from the model
  properties: {
    // Optional: any custom metadata relevant to your app
    system_prompt: systemPrompt,
    tool_call: toolName,
  },
});
```

> Prefer `begin()` -> `finish()` for new code to enable partial-event buffering,
> tracing, and automatic token counts.

---

## Tracking Signals (Feedback)

Signals capture quality ratings on AI events. Use `trackSignal()` with the same
`eventId` from `begin()` or `trackAi()`:

| Parameter   | Type                                     | Description                                 |
| ----------- | ---------------------------------------- | ------------------------------------------- |
| `eventId`   | `string`                                 | The ID of the AI event being evaluated      |
| `name`      | `"thumbs_up"`, `"thumbs_down"`, `string` | Signal name                                 |
| `type`      | `"default"`, `"feedback"`, `"edit"`      | Optional, defaults to `"default"`           |
| `comment`   | `string`                                 | User comment (for `feedback` signals)       |
| `after`     | `string`                                 | User's edited content (for `edit` signals)  |
| `sentiment` | `"POSITIVE"`, `"NEGATIVE"`               | Signal sentiment (defaults to `"NEGATIVE"`) |

```typescript
await raindrop.trackSignal({
  eventId: eventId, // The ID of the AI event being rated (from begin() or trackAi())
  name: 'thumbs_down',
  comment: userComment, // The user's feedback comment
});
```

---

## Attachments

Attachments include additional context (documents, images, code, or embedded
content) with events. Compatible with both `begin()` interactions and
`trackAi()` calls.

| Property   | Type     | Description                                   |
| ---------- | -------- | --------------------------------------------- |
| `type`     | `string` | `"code"`, `"text"`, `"image"`, or `"iframe"`  |
| `name`     | `string` | Optional display name                         |
| `value`    | `string` | Content or URL                                |
| `role`     | `string` | `"input"` or `"output"`                       |
| `language` | `string` | Programming language (for `code` attachments) |

```typescript
interaction.addAttachments([
  {
    type: 'code',
    role: 'input',
    language: 'typescript',
    name: 'example.ts',
    value: "console.log('hello');",
  },
  {
    type: 'text',
    name: 'Additional Info',
    value: 'Some extra text',
    role: 'input',
  },
  { type: 'image', value: 'https://example.com/image.png', role: 'output' },
  { type: 'iframe', value: 'https://example.com/embed', role: 'output' },
]);
```

---

## Identifying Users

```typescript
// Pass actual user data from your app's auth/session
raindrop.setUserDetails({
  userId: userId, // REQUIRED: authenticated user's unique identifier
  traits: {
    // Pass any user traits available in your app — all keys are optional and freeform
    name: userName,
    email: userEmail,
  },
});
```

---

## PII Redaction

Enable client-side PII redaction when initializing the SDK:

```typescript
new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  redactPii: true,
});
```

---

## Error Handling

Exceptions are raised when errors occur while sending events. Implement
appropriate error handling in the application.

---

## Configuration

```typescript
new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  debugLogs: process.env.NODE_ENV !== 'production', // Print queued events
  disabled: process.env.NODE_ENV === 'test', // Disable all tracking
});
```

Call `await raindrop.close()` before the process exits to flush buffered events.

---

## Tracing

Tracing captures detailed execution information from AI pipelines including
multi-model interactions, chained prompts, and tool calls.

### Getting Started

Wrap code with `withSpan` or `withTool` on an interaction. LLM calls inside are
automatically captured:

```typescript
import { Raindrop } from "raindrop-ai";

const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

const interaction = raindrop.begin({ ... });
await interaction.withSpan({ name: "my_task" }, async () => {
  // LLM calls here are automatically traced
});
```

**Next.js users:** Add `raindrop-ai` to
[`serverExternalPackages`](https://nextjs.org/docs/app/api-reference/config/next-config-js/serverExternalPackages)
in your config:

```typescript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  serverExternalPackages: ['raindrop-ai'],
};

module.exports = nextConfig;
```

### Using `withSpan`

Use `withSpan` to trace tasks or operations. Any LLM calls within the span are
automatically captured:

```typescript
// Basic span
const result = await interaction.withSpan(
  { name: 'generate_response' },
  async () => {
    return 'Generated response';
  },
);

// Span with metadata
const result = await interaction.withSpan(
  {
    name: 'embedding_generation',
    properties: { model: 'text-embedding-3-large' },
    inputParameters: ['What is the weather today?'],
  },
  async () => {
    return [0.1, 0.2, 0.3, 0.4];
  },
);
```

| Parameter         | Type                     | Description                       |
| ----------------- | ------------------------ | --------------------------------- |
| `name`            | `string`                 | Name for identification in traces |
| `properties`      | `Record<string, string>` | Additional metadata               |
| `inputParameters` | `unknown[]`              | Input parameters for the task     |

### Using `withTool`

Use `withTool` to trace agent actions - memory operations, web searches, API
calls, and more:

```typescript
// Basic tool call
const result = await interaction.withTool({ name: 'search_tool' }, async () => {
  return 'Search results';
});

// Tool with metadata
const result = await interaction.withTool(
  {
    name: 'calculator',
    properties: { operation: 'multiply' },
    inputParameters: { a: 5, b: 10 },
  },
  async () => {
    return 'Result: 50';
  },
);
```

| Parameter         | Type                     | Description                          |
| ----------------- | ------------------------ | ------------------------------------ |
| `name`            | `string`                 | Name for identification in traces    |
| `version`         | `number`                 | Version number of the tool           |
| `properties`      | `Record<string, string>` | Additional metadata                  |
| `inputParameters` | `Record<string, any>`    | Input parameters for the tool        |
| `traceContent`    | `boolean`                | Whether to trace content             |
| `suppressTracing` | `boolean`                | Suppress tracing for this invocation |

### Manual Tool Tracking

For more control over tool span tracking, use `trackTool` or `startToolSpan`.

#### `trackTool` - Retroactive Logging

Use `trackTool` to log a tool call after it has completed:

```typescript
const interaction = raindrop.begin({
  eventId: eventId,
  event: eventName, // REQUIRED: descriptive name for this AI action (e.g. 'chat_message', 'agent_run')
  userId: userId, // REQUIRED: authenticated user's unique identifier from your app
  input: userQuery, // The actual user input
});

// Log a completed tool call (pass actual tool inputs/outputs)
interaction.trackTool({
  name: 'web_search',
  input: toolInput, // The actual input passed to the tool
  output: toolOutput, // The actual output returned by the tool
  durationMs: 150,
  properties: { engine: 'google' },
});

// Log a failed tool call
interaction.trackTool({
  name: 'database_query',
  input: { query: 'SELECT * FROM users' },
  durationMs: 50,
  error: new Error('Connection timeout'),
});

interaction.finish({ output: finalResponse }); // The actual model response
```

| Parameter    | Type                     | Description                                            |
| ------------ | ------------------------ | ------------------------------------------------------ |
| `name`       | `string`                 | Name of the tool                                       |
| `input`      | `unknown`                | Input passed to the tool                               |
| `output`     | `unknown`                | Output returned by the tool                            |
| `durationMs` | `number`                 | Duration in milliseconds                               |
| `startTime`  | `Date \| number`         | When the tool started (defaults to `now - durationMs`) |
| `error`      | `Error \| string`        | Error if the tool failed                               |
| `properties` | `Record<string, string>` | Additional metadata                                    |

#### `startToolSpan` - Real-Time Tracking

Use `startToolSpan` to track a tool as it executes:

```typescript
const interaction = raindrop.begin({
  eventId: eventId,
  event: eventName, // REQUIRED: descriptive name for this AI action (e.g. 'chat_message', 'agent_run')
  userId: userId, // REQUIRED: authenticated user's unique identifier from your app
  input: userInput, // The actual user input
});

const toolSpan = interaction.startToolSpan({
  name: 'api_call',
  properties: { endpoint: '/api/data' },
  inputParameters: { method: 'GET', path: '/api/data' },
});

try {
  const result = await fetchData();
  toolSpan.setOutput(result); // Pass actual tool output
} catch (error) {
  toolSpan.setError(error);
} finally {
  toolSpan.end();
}

interaction.finish({ output: finalResponse }); // The actual model response
```

| Method              | Description                                      |
| ------------------- | ------------------------------------------------ |
| `setInput(input)`   | Set the input (JSON stringified if object)       |
| `setOutput(output)` | Set the output (JSON stringified if object)      |
| `setError(error)`   | Mark the span as failed                          |
| `end()`             | End the span (required when execution completes) |

### Module Instrumentation

If automatic instrumentation fails due to module loading order or bundler
behavior, use `instrumentModules` to explicitly specify modules to instrument:

**Anthropic users:** You must use a module namespace import (`import * as ...`),
not the default export.

```typescript
import OpenAI from 'openai';
import Anthropic from '@anthropic-ai/sdk';
import * as AnthropicModule from '@anthropic-ai/sdk'; // Required for instrumentation
import { Raindrop } from 'raindrop-ai';

const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  instrumentModules: {
    openAI: OpenAI,
    anthropic: AnthropicModule, // Pass the module namespace, not the default export
  },
});
```

Supported modules: `openAI`, `anthropic`, `cohere`, `bedrock`,
`google_vertexai`, `google_aiplatform`, `pinecone`, `together`, `langchain`,
`llamaIndex`, `chromadb`, `qdrant`, `mcp`.

### External OpenTelemetry Integration

For projects with an existing OpenTelemetry setup (Sentry, Datadog, Honeycomb,
etc.), integrate Raindrop alongside it using `useExternalOtel: true`.

**Package version:**

- If using OTEL v2: `npm install raindrop-ai@otelv2`
- If using OTEL v1 (or Sentry < 10): `npm install raindrop-ai`

#### With Sentry

```typescript
import * as Sentry from '@sentry/node';
import { Raindrop } from 'raindrop-ai';

// 1. Create Raindrop with useExternalOtel
const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  useExternalOtel: true,
});

// 2. Add Raindrop processor to Sentry.init
Sentry.init({
  dsn: process.env.SENTRY_DSN, // Your project's Sentry DSN
  tracesSampleRate: 1.0,
  openTelemetrySpanProcessors: [
    raindrop.createSpanProcessor(),
  ],
});
```

#### With Other OTEL Providers (Datadog, etc.)

> Example uses `@anthropic-ai/sdk`. Adapt imports and `instrumentModules` to the
> project's AI SDK.

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import * as AnthropicModule from '@anthropic-ai/sdk';
import { Raindrop } from 'raindrop-ai';

// 1. Create Raindrop with useExternalOtel
const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  useExternalOtel: true,
  instrumentModules: { anthropic: AnthropicModule },
});

// 2. Add Raindrop to your NodeSDK
const sdk = new NodeSDK({
  serviceName: projectName, // Replace with your app/service name
  spanProcessors: [
    raindrop.createSpanProcessor(), // Sends to Raindrop
    yourExistingProcessor, // Your existing processor
  ],
  instrumentations: raindrop.getInstrumentations(),
});
sdk.start();
```

#### Usage (Both)

> Adapt AI client import and usage to the project's AI SDK.

After setup, create AI clients and use Raindrop. For NodeSDK, create clients
**after** `sdk.start()`.

```typescript
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({ apiKey: '...' });

// LLM calls are auto-captured
// Note: input and output should be actual model I/O from your app, not placeholder strings
const interaction = raindrop.begin({
  eventId: eventId,
  event: eventName, // REQUIRED: descriptive name for this AI action
  userId: userId, // REQUIRED: authenticated user's unique identifier from your app
  input: userMessage, // The actual user input
});
const response = await interaction.withSpan({ name: 'gen' }, async () => {
  return await anthropic.messages.create({
    model: 'claude-3-haiku-20240307', // Replace with the actual model name used in your project
    max_tokens: 100,
    messages: [{ role: 'user', content: userMessage }],
  });
});
interaction.finish({ output: response.content[0].text }); // The actual model response
```

**Key methods:**

- `createSpanProcessor()` - Returns processor for NodeSDK or Sentry
- `getInstrumentations()` - Returns instrumentations for all supported libraries
  (or only those in `instrumentModules` if specified)

---

### Framework-specific guides

#### Mastra Exporter Integration Guide

When integrating Raindrop into a Mastra project as a custom
ObservabilityExporter, the following runtime behaviors must be accounted for:

1. AGENT_RUN spans are not root spans

Mastra's HTTP request handler creates a parent span before the agent execution
span. As a result, AGENT_RUN spans have isRootSpan: false and a non-empty
parentSpanId. Match on span type alone (span.type === SpanType.AGENT_RUN) and
deduplicate by traceId using a Map to prevent creating multiple interactions per
trace.

2. Span IDs are not UUIDs

Mastra uses its own internal span ID format which may not conform to UUID v4.
Always generate a fresh UUID via crypto.randomUUID() for Raindrop's eventId
parameter. Do not reuse framework-internal span or trace IDs.

3. Agent input/output is a structured messages array

**Input:** The input field on an AGENT_RUN span is not a plain string. It is an
AI SDK messages array in the format:

    [{ "role": "user", "content": [{ "type": "text", "text": "actual message" }] }]

Extract the last user message's text content before passing it to Raindrop's
input field. Handle both the string shorthand (content: "text") and the
structured content blocks format (content: [{ type: "text", text: "..." }]).

**Output:** The output field on an AGENT_RUN span is not a plain text string. It
is a JSON-serialized string containing a structured object, typically:

    '{"text":"actual assistant response","files":[]}'

Parse the JSON string and extract the text field before passing it to Raindrop's
output field. Handle parse failures gracefully by falling back to the raw
string.

4. Model name is only available on MODEL_GENERATION spans

The AGENT_RUN span does not carry model information. The model name and provider
are only available in span.attributes.model and span.attributes.provider on
MODEL_GENERATION child spans. To populate Raindrop's top-level model field in
aiData, defer the begin() call:

    * On AGENT_RUN SPAN_STARTED: buffer the trace info (input, convoId, eventId) in a pending map.
    * On MODEL_GENERATION SPAN_STARTED: materialize the pending trace by calling begin() with the model parameter from span.attributes.model.
    * On AGENT_RUN SPAN_ENDED: call finish() on the interaction. If no MODEL_GENERATION span was seen, materialize without a model as a fallback.

5. Span event ordering

Mastra emits tracing events in this order for a typical agent chat interaction:

    AGENT_RUN SPAN_STARTED
      -> PROCESSOR_RUN spans (started + ended)
      -> MODEL_GENERATION SPAN_STARTED
        -> MODEL_STEP SPAN_STARTED
          -> MODEL_CHUNK spans (streaming)
        -> MODEL_STEP SPAN_ENDED (contains usage/token counts)
      -> MODEL_GENERATION SPAN_ENDED
      -> TOOL_CALL SPAN_STARTED (if tools invoked)
      -> TOOL_CALL SPAN_ENDED
      -> (repeat MODEL_GENERATION + TOOL_CALL for multi-step agents)
      -> PROCESSOR_RUN spans (post-processing)
    AGENT_RUN SPAN_ENDED

TOOL_CALL and MCP_TOOL_CALL spans should be tracked via interaction.trackTool()
on SPAN_ENDED, when input, output, and duration are all available.

---

## Troubleshooting

- Interactions are subject to a 1 MB event limit. Oversized payloads will be
  truncated.
