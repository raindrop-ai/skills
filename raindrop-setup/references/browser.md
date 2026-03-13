---
name: browser
description: Browser SDK reference documentation
---

# Browser SDK Reference

Minimal browser-safe SDK for tracking AI events via /v1/events from web apps.

### Installation

Install the browser SDK package:

```bash
# npm
npm install @raindrop-ai/browser-sdk

# yarn
yarn add @raindrop-ai/browser-sdk

# bun
bun add @raindrop-ai/browser-sdk
```

### Quickstart

```ts
import { Raindrop } from '@raindrop-ai/browser-sdk';

const rd = new Raindrop({
  apiKey: RAINDROP_WRITE_KEY, // Your project's Raindrop write key
});

// 1) Single-shot AI event (camelCase fields)
// Note: `input` and `output` should be the actual prompt sent to and response
// received from your AI model - not literal placeholder strings.
const { eventIds } = await rd.trackAi({
  event: 'chat_message',
  userId: userId, // Required: unique user ID string (e.g. user ID, email, auth sub; or generate one if not available)
  model: modelName,
  input: userPrompt, // The actual prompt sent to the model
  output: modelResponse, // The actual response from the model
  convoId: convId, // Conversation ID — set to how your app tracks chat threads
  properties: { exampleKey: 'exampleValue' },
});
console.log('trackAi eventIds:', eventIds);

// 2) Partial AI event flow (for streaming responses)
// Pass each chunk as it arrives, then call finish() with the complete response.
const eid = uuidv4(); // import { v4 as uuidv4 } from 'uuid' — avoid Node 'crypto' module in browser/bundler environments
const partial = await rd.trackAiPartial({
  eventId: eid,
  event: 'chat_message',
  userId: userId, // Required: unique user ID string (e.g. user ID, email, auth sub; or generate one if not available)
  model: modelName,
  convoId: convId, // Conversation ID — set to how your app tracks chat threads
  output: chunk, // First streamed chunk from the model
});

await rd.trackAiPartial({ eventId: eid, output: nextChunk });
const done = await partial.finish({ output: fullResponse }); // Complete response
console.log('partial finished:', done);
```

### Identify users

```ts
// Associate traits with the current user — use actual user data from your app
await rd.identify({
  userId: userId, // The authenticated user's unique identifier
  traits: {
    name: userName,    // User's display name from your app
    email: userEmail,  // User's email from your app
    plan: userPlan,    // e.g. 'free', 'pro', 'enterprise'
  },
});

// batch — replace with actual user data from your system
await rd.identify([
  { userId: userIdA, traits: { plan: planA } },
  { userId: userIdB, traits: { plan: planB } },
]);
```

### Signals

```ts
// thumbs down with a comment
await rd.trackSignal({
  eventId: eid,
  name: 'thumbs_down',
  type: 'feedback',
  comment: 'Answer was off-topic',
});

// edit signal capturing the corrected content
await rd.trackSignal({
  eventId: eid,
  name: 'edit',
  type: 'edit',
  after: 'the corrected final text',
});
```
