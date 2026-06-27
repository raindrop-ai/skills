---
name: agno
description: Agno (Python) integration reference
---

# Agno (Python)

The `raindrop-agno` package patches an Agno `Agent` in place so every run, model call, and tool span is captured.

**Detect:** imports from `agno` (e.g. `from agno.agent import Agent`).
**Docs:** https://raindrop.ai/docs/integrations/agno

## Installation

```bash
# pip
pip install raindrop-agno

# poetry
poetry add raindrop-agno

# uv
uv add raindrop-agno
```

## Quick Start

```python
import os
from raindrop_agno import RaindropAgno
from agno.agent import Agent
from agno.models.openai import OpenAIChat

raindrop = RaindropAgno(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

agent = Agent(model=OpenAIChat(id="gpt-4o"))
raindrop.wrap(agent)  # patches agent in place

result = agent.run("Hello, world!")

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = RaindropAgno(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- `wrap` mutates the agent in place — keep using the original `agent` reference.
