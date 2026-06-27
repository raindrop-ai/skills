---
name: google-adk
description: Google ADK (Python) integration reference
---

# Google ADK (Python)

The `raindrop-google-adk` package auto-patches every `Runner` instance so every run, agent, and tool call is captured with token tracking.

**Detect:** imports from `google.adk` (e.g. `from google.adk import Runner`, `from google.adk.agents import LlmAgent`).
**Docs:** https://raindrop.ai/docs/integrations/google-adk

## Installation

```bash
# pip
pip install raindrop-google-adk

# poetry
poetry add raindrop-google-adk

# uv
uv add raindrop-google-adk
```

## Quick Start

```python
import os
from raindrop_google_adk import setup_google_adk
from google.adk import Runner
from google.adk.agents import LlmAgent
from google.adk.sessions import InMemorySessionService
from google.genai import types

raindrop = setup_google_adk(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
)  # auto-patches all Runner instances

agent = LlmAgent(
    name="assistant",
    model="gemini-2.5-flash",
    instruction="Be helpful",
)

session_service = InMemorySessionService()
runner = Runner(app_name="my_app", agent=agent, session_service=session_service)

user_msg = types.Content(
    parts=[types.Part(text="Hello, world!")],
    role="user",
)

for event in runner.run(
    user_id="user_123",  # id for end user
    session_id="session_1",
    new_message=user_msg,
):
    pass  # events flow through; telemetry captured automatically

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Projects

Route events to a specific [project](https://raindrop.ai/docs/platform/projects) by passing its slug as `project_id`:

```python
raindrop = setup_google_adk(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    project_id="support-prod",
)
```

This sets the `X-Raindrop-Project-Id` header on every event. Single-project orgs need nothing here: omit `project_id` (or pass `"default"`) to use the org's default **Production** project, which is the existing behavior. Multi-project orgs pass the target project's slug. See the [Projects docs](https://raindrop.ai/docs/platform/projects) for the full behavior table.

## Notes

- Call `setup_google_adk` once at boot — all subsequent `Runner` instances are auto-patched.
- The ADK's `user_id` / `session_id` go on `runner.run(...)`, not on the Raindrop client.
