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

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = create_raindrop_openai_agents(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- The trace processor registers automatically when the Raindrop client is constructed — no manual `add_trace_processor` needed.
