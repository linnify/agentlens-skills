# AgentLens Skills

Skills that teach coding agents (Claude Code, Cursor, Codex and others) to connect an AI agent to [AgentLens](https://agentlens.linnify.com), where domain experts review what the agent does and builders improve it from their feedback.

## The `agentlens` skill

| Section | Your coding agent can |
| --- | --- |
| Integration | Add the AgentLens SDK to a Mastra, LangChain/LangGraph, pi or OpenTelemetry agent, then run it and check the trace in AgentLens |
| Scores | Send 👍/👎 feedback, ratings and automated checks on traces |
| Reviews | Read what reviewers said about the agent, find the causes and fix them |
| Topics | File the traces that have no topic under the topics your team defined |
| Experiments | Try a change to the agent against your own dataset and compare it with the run before |
| Evaluations | Build the agent's evaluation harness, run a version on the Dataset your experts reviewed, and judge the results |
| Privacy | Keep secrets and personal data out of traces, delete a user's or tenant's traces, set retention |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills), for any coding agent it supports:

```bash
npx skills add linnify/agentlens-skills --skill agentlens
```

As a Claude Code plugin:

```
/plugin marketplace add linnify/agentlens-skills
/plugin install agentlens@agentlens-skills
```

## Before you start

- **An API key.** An owner of your AgentLens organization creates one under **Settings → API Keys**. It serves every agent of the organization. Set it as `AGENTLENS_API_KEY`.
- **An agent.** Create it under **Settings → Agents**. Its slug (`support_bot`) is how your code names it.
- **The MCP server**, so your coding agent can read traces and reviews and run experiments and evaluations: `https://agentlens.linnify.com/api/mcp`. You sign in to it with your AgentLens account, and the coding agent works with your role. The skill shows how to connect it.

## Contributing

Read [AGENTS.md](./AGENTS.md) before changing the skill.

## License

[MIT](./LICENSE)
