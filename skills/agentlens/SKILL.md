---
name: agentlens
description: >-
  Connect an AI agent to AgentLens and work with its data. Use when adding the AgentLens SDK (@linnify/agentlens-mastra, -langchain, -otel, -client, -pi) to an agent, sending scores such as 👍/👎 user feedback or automated checks, reading an agent's traces and reviews through the AgentLens MCP server to fix what reviewers flagged, or deleting an end user's traces. Use it whenever a task mentions AgentLens, its SDK, its API key or its reviews.
allowed-tools:
  - WebFetch(domain:unpkg.com)
  - mcp__agentlens__get_organization
  - mcp__agentlens__list_agents
  - mcp__agentlens__get_agent
  - mcp__agentlens__list_traces
  - mcp__agentlens__get_trace
  - mcp__agentlens__get_observation
  - mcp__agentlens__list_reviews
  - mcp__agentlens__get_review
---

# AgentLens

AgentLens is where domain experts review what an AI agent does. The agent sends its traces; experts review them and comment right on the output; each reviewed trace becomes an example in the agent's dataset; builders improve the agent and the next traces are reviewed again.

## How AgentLens is organized

| Concept | What it is |
| --- | --- |
| Organization | A team's workspace: its agents, members and API keys. |
| API key | Belongs to the organization and serves all of its agents. An owner creates it under **Settings → API Keys**; it is shown once. |
| Agent | One AI agent, named in code by its **slug**: lowercase letters, digits and underscores, up to 64 characters (`support_bot`). Every trace and score names its agent's slug, and AgentLens rejects a slug it doesn't know. |
| Trace | One run of the agent: what it was asked (input), what it answered (output) and the steps in between. Reviewers read the input and output. |
| Observation | One step of a trace: `agent` (an agent or sub-agent run), `generation` (a model call, with model, tokens and cost), `tool` (a tool call) or `span` (anything else). |
| Session | The conversation a trace belongs to. With **Multi-turn conversations** on, a session's traces are reviewed together, turn by turn. |
| User | The end user a trace ran for. Their traces can be filtered and deleted by it. |
| Tenant | The customer or workspace a trace ran for. With **Multi-tenant** on, each trace shows its tenant, with a filter and usage per tenant. |
| Environment | Where the trace was sent from: `production`, `staging`… |
| Score | A named value on a trace or one of its steps: user feedback, an automated check. |
| Review | An expert's feedback on one trace: comment threads on the output, then a decision: **Approved**, **Needs improvement** or **Drop**. |

## Connecting an agent

1. **API key.** An owner creates one under **Settings → API Keys**. It goes in the agent's environment or secret manager as `AGENTLENS_API_KEY`, and in `.env.example` without a value. Never ask the user to paste it into the chat.
2. **Agent.** Created under **Settings → Agents**, or with `save_agent` over MCP once the user confirms. Note its slug, and turn on **Multi-turn conversations** and **Multi-tenant** when they apply.
3. **SDK.** Follow [Integration](references/integration.md).

## Sections

- [Integration](references/integration.md): adding the SDK to a Mastra, LangChain/LangGraph, pi or OpenTelemetry agent, and checking its traces in AgentLens.
- [Scores](references/scores.md): user feedback and automated checks on traces.
- [Reviews](references/reviews.md): reading what reviewers said and fixing the agent.
- [Privacy](references/privacy.md): masking data, deleting a user's or tenant's traces, retention.

## Environment variables

| Variable | |
| --- | --- |
| `AGENTLENS_API_KEY` | Required. Without it the SDK sends nothing and logs a warning. |
| `AGENTLENS_BASE_URL` | Defaults to `https://api.agentlens.linnify.com`. Set it only to target another AgentLens environment. |
| `AGENTLENS_ENABLED` | `false`, `0`, `no` or `off` turns the SDK off without a warning: tests, CI, local runs. |
| `AGENTLENS_ENVIRONMENT` | The traces' environment, read by the Mastra, LangChain and pi packages. |

## SDK rules

- Install the latest release and keep every `@linnify/agentlens-*` package on the same version: they are released together.
- For an option this skill doesn't show, read the package's README for its current release: `https://unpkg.com/@linnify/agentlens-<package>/README.md`.
- A change is done when its trace or score shows up in AgentLens, not when it compiles.

## AgentLens MCP server

AgentLens serves an MCP server at `https://agentlens.linnify.com/api/mcp` (Streamable HTTP), authenticated with the organization's API key as a bearer token. When it isn't connected, offer to connect it. For Claude Code, in the project's `.mcp.json` (the key stays in the environment, so the file can be committed):

```json
{
  "mcpServers": {
    "agentlens": {
      "type": "http",
      "url": "https://agentlens.linnify.com/api/mcp",
      "headers": { "Authorization": "Bearer ${AGENTLENS_API_KEY}" }
    }
  }
}
```

Other clients take the same URL and header, with their own environment-variable syntax.

| Tool | Returns |
| --- | --- |
| `get_organization` | The organization, its members (their emails filter reviews by assignee) and its agent count |
| `list_agents`, `get_agent` | Agents with their slugs and settings. Every other tool takes the slug as `agent` |
| `list_traces` | An agent's traces, newest first, filtered by date, `sessionId`, `userId`, `environment`, `tenant` or `hasError` |
| `get_trace`, `get_observation` | One trace with its steps, scores and review; one step's full payload |
| `list_reviews`, `get_review` | Reviews by status and assignee; one review with its comment threads |
| `save_agent` | Creates or updates an agent. It writes, and a new slug sends new traces elsewhere: confirm with the user first |

Without the MCP server, the API returns an agent's latest traces with the same key:

```bash
curl -s -H "Authorization: Bearer $AGENTLENS_API_KEY" \
  "${AGENTLENS_BASE_URL:-https://api.agentlens.linnify.com}/v1/traces?agent=<slug>&limit=5"
```
