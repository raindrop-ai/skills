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

## Notes

- Pass `raindrop.handler` via `config={"callbacks": [...]}` on every `invoke` / `stream` call you want tracked.
