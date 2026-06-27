---
name: deep-agents-python
description: Deep Agents (Python) integration reference
---

# Deep Agents (Python)

The `raindrop-deep-agents` package attaches a Raindrop callback handler to LangChain Deep Agents so every planner, executor, and tool step is captured.

**Detect:** imports from `deepagents` (e.g. `from deepagents import create_deep_agent`).
**Docs:** https://raindrop.ai/docs/integrations/deepagents

## Installation

```bash
# pip
pip install raindrop-deep-agents

# poetry
poetry add raindrop-deep-agents

# uv
uv add raindrop-deep-agents
```

## Quick Start

```python
import os
from raindrop_deep_agents import RaindropDeepAgents
from deepagents import create_deep_agent
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

raindrop = RaindropDeepAgents(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

llm = ChatOpenAI(model="gpt-4o-mini")
agent = create_deep_agent(model=llm, system_prompt="Be helpful")

result = agent.invoke(
    {"messages": [HumanMessage(content="Hello, world!")]},
    config={"callbacks": [raindrop.handler]},
)

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = RaindropDeepAgents(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Pass `raindrop.handler` via `config={"callbacks": [...]}` on every `invoke` / `stream` call you want tracked.
