---
name: http-api
description: Raw HTTP API reference for integrating Raindrop when no SDK supports the target runtime.
---

# HTTP API Reference

Use this when no Raindrop SDK supports the target runtime or framework. Build a minimal client that calls these endpoints directly — do not change the app's runtime to fit an SDK.

## Before you start

Warn the user that the HTTP API is a fallback. It covers core event tracking, but lacks features that the SDKs provide out of the box:

- **Client-side PII redaction** — the SDKs can strip sensitive data before it leaves the server. With the HTTP API, you'd need to build this yourself or skip it.
- **Automatic batching and retry** — the SDKs buffer events and retry on failure. The HTTP API examples below send immediately with no retry logic.
- **Performance tuning** — the SDKs handle concurrency, backpressure, and flush timing. A raw HTTP client won't.

If the user needs these capabilities, suggest they contact support@raindrop.ai to discuss SDK support for their runtime.

---

## Base URL

```
https://api.raindrop.ai/v1/
```

## Authentication

All requests require a Bearer token in the `Authorization` header:

```
Authorization: Bearer <RAINDROP_WRITE_KEY>
Content-Type: application/json
```

---

## Endpoints

### POST `events/track`

The primary endpoint. Send AI interaction events as a JSON array (batch).

**Request body** — array of event objects:

```json
[
  {
    "event_id": "uuid-string",
    "user_id": "user-123",
    "event": "Chat Message",
    "timestamp": "2026-03-11T14:30:45",
    "ai_data": {
      "model": "gpt-4o",
      "input": "What is the weather?",
      "output": "It's sunny and 72°F.",
      "convo_id": "conv-abc"
    },
    "properties": {},
    "attachments": []
  }
]
```

**Required fields:**
- `event_id` — UUID string. Generate one per event.
- `user_id` — identifies the end user.
- `event` — name of the interaction (e.g., `"Chat Message"`).
- `ai_data` — must contain at least one of `input` or `output`.

**Optional fields:**
- `timestamp` — ISO 8601, UTC, no microseconds. Defaults to server time.
- `ai_data.model` — the LLM model name.
- `ai_data.convo_id` — groups events into a conversation thread.
- `properties` — arbitrary key-value metadata.
- `attachments` — array of attachment objects (see below).

**Size limit:** 1 MB per event.

---

### POST `events/track_partial`

For streaming interactions. Sent as a **single object** (not an array). Multiple partial events with the same `event_id` are merged server-side.

```json
{
  "event_id": "uuid-string",
  "user_id": "user-123",
  "event": "Chat Message",
  "ai_data": {
    "output": "partial response so far..."
  },
  "is_pending": true
}
```

Send a final update with `"is_pending": false` (or omit it) and the complete `output` to finalize.

---

### POST `users/identify`

Identify users with traits. Batched as an array.

```json
[
  {
    "user_id": "user-123",
    "traits": {
      "name": "Jane Doe",
      "plan": "pro"
    }
  }
]
```

---

### POST `signals/track`

Track feedback, edits, or custom signals on existing events. Batched as an array.

```json
[
  {
    "event_id": "uuid-of-original-event",
    "signal_name": "thumbs_up",
    "timestamp": "2026-03-11T14:31:00",
    "signal_type": "feedback",
    "properties": {
      "comment": "Great answer!"
    },
    "sentiment": "POSITIVE"
  }
]
```

**Signal types:**
- `"feedback"` — requires `properties.comment` (non-empty string). Optionally set `sentiment` to `"POSITIVE"` or `"NEGATIVE"`.
- `"edit"` — requires `properties.after` (the edited content).
- `"default"` — no special requirements.

Optional: `attachment_id` to associate the signal with a specific attachment.

---

## Attachments

Attach code, text, images, or iframes to an event:

```json
{
  "type": "code",
  "value": "console.log('hello')",
  "name": "example.js",
  "role": "output",
  "language": "javascript"
}
```

- `type` — one of `"code"`, `"text"`, `"image"`, `"iframe"`.
- `value` — the content (raw code, text, URL for images/iframes).
- `name` — optional display name.
- `role` — optional: `"input"`, `"output"`, or `"context"`.
- `language` — optional, for code snippets.

---

## Minimal Client Example

A single `fetch`-based function is all you need:

```typescript
const RAINDROP_URL = "https://api.raindrop.ai/v1/";

async function trackEvent(
  writeKey: string,
  event: {
    event_id: string;
    user_id: string;
    event: string;
    ai_data: { model?: string; input?: string; output?: string; convo_id?: string };
    properties?: Record<string, unknown>;
  }
) {
  await fetch(`${RAINDROP_URL}events/track`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${writeKey}`,
    },
    body: JSON.stringify([event]),
  });
}
```

For serverless/edge: always `await` the fetch (or use `waitUntil` if available) — fire-and-forget will lose events when the runtime shuts down.

---

## Buffering (Optional)

The SDKs buffer events and flush every ~1 second in batches of 10. For a minimal client, you can either:

1. **Send immediately** — simplest, fine for low-volume use.
2. **Buffer and flush** — accumulate events in an array and POST them periodically or on a size threshold. Flush before the process/request exits.

---

## Retries

On network failure, retry up to 3 times. The API is idempotent on `event_id`, so duplicate sends are safe.
