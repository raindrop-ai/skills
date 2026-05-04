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

## Notes

- `wrap` mutates the agent in place — keep using the original `agent` reference.
