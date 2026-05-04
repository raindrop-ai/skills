---
name: dspy
description: DSPy (Python) integration reference
---

# DSPy (Python)

The `raindrop-dspy` package wraps a DSPy module so every `Predict` / `ChainOfThought` call is captured.

**Detect:** imports from `dspy` (e.g. `import dspy`).
**Docs:** https://raindrop.ai/docs/integrations/dspy

## Installation

```bash
# pip
pip install raindrop-dspy

# poetry
poetry add raindrop-dspy

# uv
uv add raindrop-dspy
```

## Quick Start

```python
import os
import dspy
from raindrop_dspy import RaindropDSPy

raindrop = RaindropDSPy(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

lm = dspy.LM("openai/gpt-4o-mini")
dspy.configure(lm=lm)

predict = dspy.Predict("question -> answer")
wrapped = raindrop.wrap(predict)

result = wrapped(question="Hello, world!")

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Notes

- `wrap` returns a wrapped module — call the wrapped reference, not the original.
