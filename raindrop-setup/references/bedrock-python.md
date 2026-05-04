---
name: bedrock-python
description: AWS Bedrock (Python) integration reference
---

# AWS Bedrock (Python)

The `raindrop-bedrock` package patches a `bedrock-runtime` boto3 client in place so every `converse` and `invoke_model` call is captured.

**Detect:** `boto3.client("bedrock-runtime", ...)` calls, or imports from `boto3` paired with Bedrock model IDs.
**Docs:** https://raindrop.ai/docs/integrations/bedrock

## Installation

```bash
# pip
pip install raindrop-bedrock

# poetry
poetry add raindrop-bedrock

# uv
uv add raindrop-bedrock
```

## Quick Start

```python
import os
import boto3
from raindrop_bedrock import RaindropBedrock

raindrop = RaindropBedrock(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

client = boto3.client("bedrock-runtime", region_name="us-east-1")
raindrop.wrap(client)  # patches client in place

result = client.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[{"role": "user", "content": [{"text": "Hello, world!"}]}],
)

# ...

raindrop.flush()  # publish queued events before shutdown
```

## Notes

- `wrap` mutates the client in place — continue using the original `client` reference.
