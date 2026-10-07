# Raindrop Skills

Open-source skills for integrating [Raindrop](https://raindrop.ai) into your AI projects. These work with any AI coding assistant that supports the skills format.

## Install

```bash
npx skills add raindrop-ai/skills --skill raindrop-setup
npx skills add raindrop-ai/skills --skill raindrop-investigate
npx skills add raindrop-ai/skills --skill raindrop-evals
npx skills add raindrop-ai/skills --skill raindrop-ai-sdk-v7
```

## Available skills

| Skill | Description |
|-------|-------------|
| [`raindrop-setup`](./raindrop-setup/) | Set up, debug, or extend Raindrop observability in a project. |
| [`raindrop-investigate`](./raindrop-investigate/) | Investigate AI application issues and refine signals using Raindrop's MCP tools. |
| [`raindrop-ai-sdk-v7`](./raindrop-ai-sdk-v7/) | Integrate Raindrop with Vercel AI SDK v7 (beta/canary) and fix v7-specific telemetry gaps. |

For local application evals, use [`raindrop-evals`](./raindrop-evals/). Your agent can
reuse saved datasets and evaluators, create a run manifest, and run it with Vitest.

## About Raindrop

Raindrop is monitoring for AI agents. Track behavior, detect issues before your users do, and understand what your agent is actually doing in production.

Learn more at [raindrop.ai](https://raindrop.ai).
