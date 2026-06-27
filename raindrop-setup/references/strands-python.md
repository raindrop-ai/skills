---
name: strands-python
description: Strands Agents (Python) integration reference
---

# Strands Agents (Python)

The `raindrop-strands` package registers hooks on a Strands `Agent` so every run and tool call is captured.

**Detect:** imports from `strands` (e.g. `from strands import Agent`).
**Docs:** https://raindrop.ai/docs/integrations/strands

## Installation

```bash
# pip
pip install raindrop-strands

# poetry
poetry add raindrop-strands

# uv
uv add raindrop-strands
```

## Quick Start

```python
import os
from strands import Agent
from raindrop_strands import RaindropStrands

raindrop = RaindropStrands(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

agent = Agent(
    model="us.amazon.nova-lite-v1:0",
    system_prompt="Be helpful",
)

raindrop.handler.register_hooks(agent)

result = agent("Hello, world!")

# ...

raindrop.flush()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = RaindropStrands(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- `register_hooks` mutates the agent in place — call it once after creation.
