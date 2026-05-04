---
name: azure-openai-python
description: Azure OpenAI (Python) integration reference
---

# Azure OpenAI (Python)

The `raindrop-azure-openai` package patches an `AzureOpenAI` client in place so every chat completion is captured.

**Detect:** `AzureOpenAI` imported from `openai`, or `AZURE_OPENAI_*` environment variables in use.
**Docs:** https://raindrop.ai/docs/integrations/azure-openai

## Installation

```bash
# pip
pip install raindrop-azure-openai

# poetry
poetry add raindrop-azure-openai

# uv
uv add raindrop-azure-openai
```

## Quick Start

```python
import os
from raindrop_azure_openai import RaindropAzureOpenAI
from openai import AzureOpenAI

raindrop = RaindropAzureOpenAI(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_KEY"],
    api_version="2024-10-21",
)
raindrop.wrap(client)  # patches client in place

result = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello, world!"}],
)

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Notes

- `wrap` mutates the client in place — keep using the original `client` reference.
