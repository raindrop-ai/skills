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

## Notes

- `register_hooks` mutates the agent in place — call it once after creation.
