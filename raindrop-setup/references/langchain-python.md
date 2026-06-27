---
name: langchain-python
description: LangChain (Python) integration reference
---

# LangChain (Python)

The `raindrop-langchain` package attaches a Raindrop callback handler to LangChain and LangGraph runs so every chain, model, and tool step is captured.

**Detect:** imports from `langchain`, `langchain_core`, `langchain_openai`, `langchain_anthropic`, or `langgraph`.
**Docs:** https://raindrop.ai/docs/integrations/langchain

## Installation

```bash
# pip
pip install raindrop-langchain

# poetry
poetry add raindrop-langchain

# uv
uv add raindrop-langchain
```

## Quick Start

```python
import os
from raindrop_langchain import create_raindrop_langchain
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

raindrop = create_raindrop_langchain(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

model = ChatOpenAI(model="gpt-4o")

result = model.invoke(
    [HumanMessage(content="Hello, world!")],
    config={"callbacks": [raindrop.handler]},
)

# ...

raindrop.flush()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = create_raindrop_langchain(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Pass `raindrop.handler` via `config={"callbacks": [...]}` on every invoke / stream call you want tracked.
