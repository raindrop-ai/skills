---
name: go
description: Go SDK reference documentation
---

# Go SDK Reference

The `github.com/raindrop-ai/go` package is the base Go SDK. There are no framework-specific Go integrations today — wrap your AI calls with `Begin()` / `Finish()` directly.

## Installation

```bash
go get github.com/raindrop-ai/go
```

## Quick Start

Construct the client once at boot, then wrap each AI call. Add spans for nested work to build a trace.

```go
package main

import (
    "context"
    "log"
    "os"

    raindrop "github.com/raindrop-ai/go"
)

func main() {
    raindropClient, err := raindrop.New(
        raindrop.WithWriteKey(os.Getenv("RAINDROP_WRITE_KEY")),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer raindropClient.Close()

    ctx := context.Background()
    prompt := "Hello, world!"

    interaction := raindropClient.Begin(ctx, raindrop.BeginOptions{
        Event:   "chat",
        Model:   "gpt-4o",
        Input:   prompt,
        UserID:  "user_123",   // id for end user
        ConvoID: "convo_456",  // id for conversation
    })

    _ = interaction.WithSpan(
        raindrop.SpanOptions{Name: "retrieve_docs"},
        func(ctx context.Context, s *raindrop.Span) error {
            // your retrieval / tool work
            return nil
        },
    )

    result := callYourModel(prompt)

    _ = interaction.Finish(raindrop.FinishOptions{Output: result})
}
```

## Key concepts

- **`Begin()` → `Finish()`** brackets a single AI interaction. Both `Event` and `UserID` are required.
- **`WithSpan`** captures nested work (retrieval, tool calls, chained prompts) so the trace shows the structure.
- **`Close()`** must run before process exit — typically via `defer` — to flush queued events. Skipping it drops trailing events.

## Auxiliary tracking

- **User identity** — set `UserID` on `BeginOptions`. If you have an email or display name, capture them as event properties.
- **Conversation threading** — set `ConvoID` on `BeginOptions` to group multi-turn flows.
- **Properties** — pass app-specific metadata via `BeginOptions.Properties` (map). Don't put `model` / `input` / `output` / `convoId` here — they have first-class fields.

## Serverless / short-lived processes

`defer raindropClient.Close()` is not enough for AWS Lambda / Cloud Run handlers — the runtime can freeze the process before deferred functions complete. Call `Close()` (or its non-blocking flush variant if available in your version) explicitly before the handler returns.

## Documentation

Latest: https://raindrop.ai/docs/sdk/go
