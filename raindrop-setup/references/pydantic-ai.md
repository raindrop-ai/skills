---
name: pydantic-ai
description: Pydantic AI (Python) integration reference
---

# Pydantic AI (Python)

The `raindrop-pydantic-ai` package patches a Pydantic AI `Agent` in place so every run and tool call is captured.

**Detect:** imports from `pydantic_ai` (e.g. `from pydantic_ai import Agent`).
**Docs:** https://raindrop.ai/docs/integrations/pydantic-ai

## Installation

```bash
# pip
pip install raindrop-pydantic-ai

# poetry
poetry add raindrop-pydantic-ai

# uv
uv add raindrop-pydantic-ai
```

## Quick Start

```python
import os
from raindrop_pydantic_ai import RaindropPydanticAI
from pydantic_ai import Agent

raindrop = RaindropPydanticAI(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

agent = Agent("openai:gpt-4o", system_prompt="Be helpful")
raindrop.wrap(agent)  # patches agent in place

result = agent.run_sync("Hello, world!")

# ...

raindrop.flush()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = RaindropPydanticAI(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- `wrap` mutates the agent in place — no need to use a separate wrapped object.
