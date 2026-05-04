---
name: openai-agents-python
description: OpenAI Agents SDK (Python) integration reference
---

# OpenAI Agents SDK (Python)

The `raindrop-openai-agents` package auto-registers a Raindrop trace processor so every Agents SDK run, handoff, and tool call is captured.

**Detect:** imports from `agents` (e.g. `from agents import Agent, Runner`).
**Docs:** https://raindrop.ai/docs/integrations/openai-agents

## Installation

```bash
# pip
pip install raindrop-openai-agents

# poetry
poetry add raindrop-openai-agents

# uv
uv add raindrop-openai-agents
```

## Quick Start

```python
import os
from raindrop_openai_agents import create_raindrop_openai_agents
from agents import Agent, Runner

raindrop = create_raindrop_openai_agents(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)
# integration is auto-registered; no extra setup needed

agent = Agent(name="Assistant", model="gpt-4o", instructions="Be helpful")

result = Runner.run_sync(agent, "Hello, world!")

# ...

raindrop.flush()  # publish queued events before shutdown
```

## Notes

- The trace processor registers automatically when the Raindrop client is constructed — no manual `add_trace_processor` needed.
