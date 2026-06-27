---
name: vertex-ai-python
description: Vertex AI / Google Gen AI (Python) integration reference
---

# Vertex AI (Python)

The `raindrop-vertex-ai` package patches a `genai.Client` in place so every `generate_content` call is captured.

**Detect:** imports from `google.genai` (e.g. `from google import genai`).
**Docs:** https://raindrop.ai/docs/integrations/vertex-ai

## Installation

```bash
# pip
pip install raindrop-vertex-ai

# poetry
poetry add raindrop-vertex-ai

# uv
uv add raindrop-vertex-ai
```

## Quick Start

```python
import os
from raindrop_vertex_ai import RaindropVertexAI
from google import genai

raindrop = RaindropVertexAI(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

client = genai.Client(api_key=os.environ["GOOGLE_API_KEY"])
raindrop.wrap(client)  # patches client in place

result = client.models.generate_content(
    model="gemini-2.0-flash",
    contents="Hello, world!",
)

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = RaindropVertexAI(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- `wrap` mutates the client in place; continue using the original `client` reference.
