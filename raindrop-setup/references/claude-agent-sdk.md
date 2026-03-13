---
name: claude-agent-sdk
description: Claude Agent SDK reference documentation
---

# Claude Agent SDK Reference

The `@raindrop-ai/claude-agent-sdk` package wraps the
[Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) so
you can capture events and traces from `query()` calls. Pass `eventMetadata()`
to the queries you want tracked in Raindrop.

## Installation

```bash
# npm
npm install @raindrop-ai/claude-agent-sdk

# yarn
yarn add @raindrop-ai/claude-agent-sdk

# pnpm
pnpm add @raindrop-ai/claude-agent-sdk

# bun
bun add @raindrop-ai/claude-agent-sdk
```

## Quick Start

```typescript
import * as claudeAgentSDK from '@anthropic-ai/claude-agent-sdk';
import {
  createRaindropClaudeAgentSDK,
  eventMetadata,
} from '@raindrop-ai/claude-agent-sdk';

// 1. Create the Raindrop client
const raindrop = createRaindropClaudeAgentSDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

// 2. Wrap the Claude Agent SDK
const { query, createSdkMcpServer, ReadTool, EditTool } = raindrop.wrap(
  claudeAgentSDK,
  {
    selfDiagnostics: { enabled: true },
  },
);

// 3. Add eventMetadata() on queries you want tracked
for await (const message of query(
  {
    prompt: 'List files in the current directory',
    options: {
      allowedTools: ['Bash'],
      model: 'claude-sonnet-4.5',
    },
  },
  eventMetadata({
    userId, // REQUIRED: authenticated user id from your app
    eventName, // Optional: e.g. 'agent_query'
  }),
)) {
  console.log(message);
}

// Optional: identify user traits
await raindrop.users.identify({
  userId,
  traits: {
    plan: userPlan,
    email: userEmail,
  },
});

// 4. Flush before shutdown (serverless/scripts)
await raindrop.flush();
```

Queries with `eventMetadata()` are tracked with events and traces. Queries
without it run normally and are not tracked.

---

## Configuration

### Client Options

```typescript
const raindrop = createRaindropClaudeAgentSDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!, // REQUIRED
  endpoint: 'https://api.raindrop.ai/v1', // Optional, defaults to production

  traces: {
    enabled: true, // Default: true
    flushIntervalMs: 5000, // Default: 5000
    maxBatchSize: 100, // Default: 100
    maxQueueSize: 1000, // Default: 1000
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
const wrapped = raindrop.wrap(claudeAgentSDK, {
  selfDiagnostics: { enabled: true },
});
```

### Self Diagnostics

Enabling `selfDiagnostics` injects a tool that allows the users' AI to report
issues it detects during execution (for example: tool failures, looping
behavior, or missing capabilities). The tool is delivered as an in-process MCP
server, so it works alongside any other MCP servers the user configures. The
tool name defaults to `__raindrop_report`, and can be overridden with
`selfDiagnostics.toolName`. The SDK automatically prompt-engineers the tool
description from the developer provided `signals` definitions;
`selfDiagnostics.guidance` is optional extra guidance.

```typescript
import * as claudeAgentSDK from '@anthropic-ai/claude-agent-sdk';
import {
  createRaindropClaudeAgentSDK,
  eventMetadata,
} from '@raindrop-ai/claude-agent-sdk';

const raindrop = createRaindropClaudeAgentSDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
});

const wrapped = raindrop.wrap(claudeAgentSDK, {
  selfDiagnostics: {
    enabled: true,
    // Optional: replace defaults with domain-specific categories
    signals: {
      missing_context: {
        description:
          'You cannot complete the task because critical information, credentials, or access is missing and the user cannot provide it. ' +
          'Do NOT report this for normal clarifying questions — only when you are blocked.',
        sentiment: 'NEGATIVE',
      },
      repeatedly_broken_tool: {
        description:
          'A tool has failed or not returned the expected response on multiple distinct attempts in this conversation, preventing task completion. ' +
          'A single tool error is NOT enough — the tool must be persistently broken or aberrantly behaving across retries.',
        sentiment: 'NEGATIVE',
      },
      complete_task_failure: {
        description:
          'You were unable to accomplish what the user asked despite making genuine attempts. ' +
          'This is NOT a refusal or policy block — you tried and failed to deliver the result.',
        sentiment: 'NEGATIVE',
      },
    },
  },
});
```

When invoked, the injected tool tracks a signal on the same `eventId`:

```json
{
  "event_id": "evt_...",
  "signal_name": "agent:tool_failure",
  "signal_type": "agent",
  "properties": {
    "source": "agent_reporting_tool",
    "category": "tool_failure",
    "signal_description": "A tool call returned an error, timed out, or produced unusable output.",
    "detail": "The search API returned a 503 after two retries."
  }
}
```

The self-diagnostics tool is invisible: it is auto-approved in the PreToolUse
hook and its tool_use/tool_result blocks are stripped from the SDK stream, so
the user never sees it and doesn't need to handle it.

## Per-Query Context

Use `eventMetadata()` as the second argument when you want a query tracked.
`userId` is required:

```typescript
import { eventMetadata } from '@raindrop-ai/claude-agent-sdk';

const wrapped = raindrop.wrap(claudeAgentSDK, {
  selfDiagnostics: { enabled: true },
});

for await (const msg of wrapped.query(
  { prompt: userPrompt, options: { model: modelName } },
  eventMetadata({
    userId, // REQUIRED
    convoId, // Optional: conversation/thread id from your app
    eventName, // Optional
    properties: { tier: userTier },
  }),
)) {
  console.log(msg);
}
```

You can also set `convoId`, `eventName`, `properties`, or `eventId` per query.

**Defaults:**

- If `eventMetadata()` is omitted, the query runs normally but no events or
  traces are captured by Raindrop
- `eventName` defaults to `"agent_query"`
- `convoId` is auto-captured from Claude's `session_id` if not provided
- `eventId` is auto-generated if not provided

---

## Identifying Users

Use `users.identify` to associate traits with a user:

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

## Signals (Feedback)

Track user feedback on AI responses using the same `eventId`:

```typescript
import { eventMetadata } from '@raindrop-ai/claude-agent-sdk';

const eventId = crypto.randomUUID();
const wrapped = raindrop.wrap(claudeAgentSDK, {
  selfDiagnostics: { enabled: true },
});

for await (const msg of wrapped.query(
  { prompt: userMessage, options: { model: 'claude-sonnet-4.5' } },
  eventMetadata({ userId, eventId }),
)) {
  console.log(msg);
}

// Later, when user gives feedback
await raindrop.signals.track({
  eventId,
  name: 'thumbs_down',
  comment: 'Answer was incorrect',
});
```

### Signal Types

| Type         | Use Case                         |
| ------------ | -------------------------------- |
| `"default"`  | Generic signals (thumbs up/down) |
| `"feedback"` | User comments about quality      |
| `"edit"`     | User corrected the output        |

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

Update events after they're created (use `eventId` from `eventMetadata`):

```typescript
const eventId = crypto.randomUUID();
const wrapped = raindrop.wrap(claudeAgentSDK, {
  selfDiagnostics: { enabled: true },
});

for await (const msg of wrapped.query(
  { prompt, options },
  eventMetadata({ userId, eventId }),
)) {
  // ...
}

// Add properties after the query completes
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

// Mark as complete (triggers immediate shipping)
await raindrop.events.finish(eventId);
```

The event is automatically finalized when the query stream completes. Use
`finish()` only when you need to force immediate shipping before the stream
ends.

---

## Flush & Shutdown

Always flush before your process exits to ensure all data is sent:

```typescript
import { eventMetadata } from '@raindrop-ai/claude-agent-sdk';

// Serverless: flush at end of request
export async function POST(request: Request) {
  const wrapped = raindrop.wrap(claudeAgentSDK, {
    selfDiagnostics: { enabled: true },
  });
  const messages = [];
  for await (const msg of wrapped.query(
    { prompt, options },
    eventMetadata({ userId }),
  )) {
    messages.push(msg);
  }
  await raindrop.flush();
  return Response.json({ messages });
}

// Long-running: shutdown gracefully
process.on('SIGTERM', async () => {
  await raindrop.shutdown();
  process.exit(0);
});
```

---

## Debugging

Enable debug logging to troubleshoot issues:

```typescript
const raindrop = createRaindropClaudeAgentSDK({
  writeKey: process.env.RAINDROP_WRITE_KEY!,
  events: { debug: true },
  traces: { debug: true, debugSpans: true },
});
```

Or via environment variable:

```bash
RAINDROP_AI_DEBUG=1 node your-script.js
```

This logs:

- Every event sent to Raindrop
- Every trace batch shipped
- Span parent/child relationships (with `debugSpans`)

---

## Troubleshooting

### Events not appearing in dashboard

1. **Ensure tracked queries include `eventMetadata()`**
2. **Check your write key** - Ensure `RAINDROP_WRITE_KEY` is set correctly
3. **Flush before exit** - Call `await raindrop.flush()` before your process
   ends
4. **Enable debug logging** - Set `events: { debug: true }` to see what's being
   sent

### Traces missing or incomplete

1. **Enable trace debugging** - Set `traces: { debug: true, debugSpans: true }`
2. **Check for errors** - Look for `[raindrop-ai/claude-agent-sdk]` prefixed
   logs

### Wrapper not capturing tool calls

Ensure the wrapped `query` is the one you call. Tools are traced via SDK hooks;
if you call the unwrapped SDK directly, no tracing occurs.
