---
name: crewai
description: CrewAI (Python) integration reference
---

# CrewAI (Python)

The `raindrop-crewai` package patches a `Crew` in place so every `kickoff` is captured with task and token tracking.

**Detect:** imports from `crewai` (e.g. `from crewai import Agent, Crew, Task`).
**Docs:** https://raindrop.ai/docs/integrations/crewai

## Installation

```bash
# pip
pip install raindrop-crewai

# poetry
poetry add raindrop-crewai

# uv
uv add raindrop-crewai
```

## Quick Start

```python
import os
from raindrop_crewai import RaindropCrewAI
from crewai import Agent, Crew, Task

raindrop = RaindropCrewAI(
    api_key=os.environ["RAINDROP_WRITE_KEY"],
    user_id="user_123",  # id for end user
    convo_id="convo_456",  # id for conversation
)

agent = Agent(
    role="Senior Researcher",
    goal="Find one interesting fact about {topic}",
    backstory="You are an experienced researcher.",
)
task = Task(
    description="Identify one interesting fact about {topic}.",
    expected_output="A single sentence fact.",
    agent=agent,
)
crew = Crew(agents=[agent], tasks=[task])
raindrop.wrap(crew)  # patches crew in place

result = crew.kickoff(inputs={"topic": "Hello, world!"})

# ...

raindrop.shutdown()  # publish queued events before shutdown
```

## Notes

- Wrap each `Crew` per conversation so `convo_id` lines up with the user-facing thread.
