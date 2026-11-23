---
name: agentlens
description: >-
  Connect an AI agent to AgentLens and work with its data. Use when adding the AgentLens SDK (@linnify/agentlens-mastra, -langchain, -otel, -client, -pi) to an agent, sending scores such as 👍/👎 user feedback or automated checks, reading an agent's traces and reviews through the AgentLens MCP server to fix what reviewers flagged, filing traces under topics, trying a change to the agent against a local dataset as an experiment, building or running an evaluation harness and judging its results, or deleting an end user's traces. Use it whenever a task mentions AgentLens, its SDK, its API key, its reviews, its experiments or its evaluations.
allowed-tools:
  - WebFetch(domain:unpkg.com)
  - mcp__agentlens__list_organizations
  - mcp__agentlens__list_agents
  - mcp__agentlens__get_agent
  - mcp__agentlens__list_traces
  - mcp__agentlens__get_trace
  - mcp__agentlens__get_observation
  - mcp__agentlens__list_reviews
  - mcp__agentlens__get_review
  - mcp__agentlens__list_topics
  - mcp__agentlens__list_versions
  - mcp__agentlens__list_dataset_items
  - mcp__agentlens__get_dataset_item
  - mcp__agentlens__list_evaluations
  - mcp__agentlens__get_evaluation
  - mcp__agentlens__get_evaluation_result
  - mcp__agentlens__list_experiments
  - mcp__agentlens__get_experiment
  - mcp__agentlens__list_experiment_items
  - mcp__agentlens__compare_experiments
---

# AgentLens

AgentLens is where domain experts review what an AI agent does. The agent sends its traces; experts review them and comment right on the output; each reviewed trace becomes an item in the agent's Dataset; builders improve the agent, check the change against that Dataset with an evaluation, and the next traces are reviewed again.

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
| Topic | A kind of request the agent handles, with rules that say what belongs to it. A trace is filed under one topic or none. |
| Review | An expert's feedback on one trace: comment threads on the output, then a decision: **Approved**, **Needs improvement** or **Drop**. |
| Dataset item | A reviewed trace kept as a case: **Approved** when its output is the expected answer, **Open** when the review flagged its output, **Dropped** when it no longer runs. |
| Version | A named state of the agent (`v1.1`) with a note on what changed. Its configuration lives in the agent's repository, not in AgentLens. |
| Evaluation | One run of a version on the Dataset, executed by the agent's own **harness**. An approved item passes when the new answer is the same as the expected one, an open item when the new answer fixes what the review flagged. |
| Experiment | One run of the project's own local dataset against a change to the agent, started from a script. Each case is an item with its input, context, trace and scores. |

Traces of experiments and evaluations stay out of the normal trace lists; they are reached through their experiment or evaluation.

## Connecting an agent

1. **API key.** An owner creates one under **Settings → API Keys**. It goes in the agent's environment or secret manager as `AGENTLENS_API_KEY`, and in `.env.example` without a value. Never ask the user to paste it into the chat.
2. **Agent.** Created under **Settings → Agents**, or with `save_agent` over MCP once the user confirms. Note its slug, and turn on **Multi-turn conversations** and **Multi-tenant** when they apply.
3. **SDK.** Follow [Integration](references/integration.md).

## Sections

- [Integration](references/integration.md): adding the SDK to a Mastra, LangChain/LangGraph, pi or OpenTelemetry agent, and checking its traces in AgentLens.
- [Scores](references/scores.md): user feedback and automated checks on traces.
- [Reviews](references/reviews.md): reading what reviewers said and fixing the agent.
- [Topics](references/topics.md): filing traces that have no topic.
- [Experiments](references/experiments.md): trying a change to the agent against the project's local dataset and comparing it with the run before.
- [Evaluations](references/evaluations.md): building the agent's evaluation harness, running a version on the Dataset, judging the results the harness left undecided.
- [Privacy](references/privacy.md): masking data, deleting a user's or tenant's traces, retention.

## Environment variables

| Variable | |
| --- | --- |
| `AGENTLENS_API_KEY` | Required. Without it the SDK sends nothing and logs a warning. |
| `AGENTLENS_BASE_URL` | The AgentLens web app's address. Defaults to `https://agentlens.linnify.com`; set it only for another AgentLens, such as a local one (`http://localhost:3000`). The SDK adds `/api/v1` itself, so never end it in `/api` or `/v1`. The old api address, `https://api.agentlens.linnify.com`, answers only for releases before 0.9.0: remove it wherever a project still sets it. |
| `AGENTLENS_ENABLED` | `false`, `0`, `no` or `off` turns the SDK off without a warning: tests, CI, local runs. |
| `AGENTLENS_ENVIRONMENT` | The traces' environment, read by the Mastra, LangChain and pi packages. |

## SDK rules

- Install the latest release and keep every `@linnify/agentlens-*` package on the same version: they are released together. Releases before 0.9.0 still deliver traces and scores, through the old api address, but have no experiments or evaluations: upgrade a project that pins one.
- For an option this skill doesn't show, read the package's README for its current release: `https://unpkg.com/@linnify/agentlens-<package>/README.md`.
- A change is done when its trace or score shows up in AgentLens, not when it compiles.

## AgentLens MCP server

AgentLens serves an MCP server at `https://agentlens.linnify.com/api/mcp` (Streamable HTTP). It signs the user in with OAuth, and you then act as that user: you see the agents they can open, with their role on each. When it isn't connected, offer to connect it. For Claude Code:

```bash
claude mcp add --transport http --scope user agentlens https://agentlens.linnify.com/api/mcp
```

The user then runs `/mcp`, picks `agentlens` and chooses **Authenticate**. When their browser is on another machine than Claude Code, `claude mcp login agentlens --no-browser` prints the sign-in link and takes the redirect URL back. Other clients add the same URL as a remote HTTP server.

A tool refuses what the user's role cannot do in the web app, with the role it needs:

| Role | Can |
| --- | --- |
| Expert | Read reviews |
| Builder | Everything an Expert can, plus traces, topics, the Dataset, versions, evaluations and experiments |
| Owner | Everything a Builder can, plus deciding evaluation results, moving Dataset items, and an agent's slug, retention and harness URL |

Every tool but the first two takes the agent's slug as `agent`.

| Tools | For |
| --- | --- |
| `list_agents`, `list_organizations`, `get_agent` | The agents the user can open with their slugs, settings and members (their emails filter reviews by assignee) |
| `list_traces` | An agent's traces, newest first, filtered by date, `name`, `sessionId`, `userId`, `environment`, `tenant`, `hasError`, `topic` (an id, or `"none"`) or a score |
| `get_trace`, `get_observation` | One trace with its steps, topic, scores and review; one step's full payload |
| `list_reviews`, `get_review` | Reviews by status and assignee; one review with its comment threads |
| `list_topics`, `set_trace_topic` | [Topics](references/topics.md) |
| `create_experiment`, `update_experiment`, `list_experiments`, `get_experiment`, `list_experiment_items`, `compare_experiments`, `delete_experiment` | [Experiments](references/experiments.md) |
| `list_versions`, `create_version`, `list_dataset_items`, `get_dataset_item`, `list_evaluations`, `get_evaluation`, `get_evaluation_result`, `start_evaluation`, `decide_evaluation_result`, `send_evaluation_result_to_expert`, `approve_dataset_item`, `reopen_dataset_item`, `drop_dataset_items` | [Evaluations](references/evaluations.md) |
| `save_agent` | Creates or updates an agent, including its harness URL |

Tools that write change what the whole team sees. Confirm with the user before `save_agent` (a new slug sends new traces elsewhere), `start_evaluation`, `create_version`, `decide_evaluation_result`, `send_evaluation_result_to_expert`, the three Dataset item tools and `delete_experiment`. `create_experiment`, `update_experiment` and `set_trace_topic` are part of the tasks that use them.

Without the MCP server, AgentLens returns an agent's latest traces with the API key:

```bash
curl -s -H "Authorization: Bearer $AGENTLENS_API_KEY" \
  "${AGENTLENS_BASE_URL:-https://agentlens.linnify.com}/api/v1/agents/<slug>/traces?limit=5"
```
