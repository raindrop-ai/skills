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

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = RaindropDSPy(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- `wrap` returns a wrapped module — call the wrapped reference, not the original.
