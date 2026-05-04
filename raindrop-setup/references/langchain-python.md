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

## Notes

- Pass `raindrop.handler` via `config={"callbacks": [...]}` on every invoke / stream call you want tracked.
