# Privacy

## What gets sent

Every input, output and metadata value of every step is stored in AgentLens and shown to everyone who can open the agent, reviewers included.

- Keep credentials, tokens and API keys out entirely. Graph state and tool arguments are where they usually leak.
- Mask personal data reviewers don't need to judge the answer. The LangChain handler takes a `mask` function; with the other packages, strip it before it becomes a span's input, output or metadata.
- Set the user on every trace that belongs to an end user: it is how their traces are found and deleted later.

## Deleting traces

```ts
const deleted = await client.trace.deleteMany({ userId: 'user_123' });
await client.trace.deleteMany({ agent: 'billing_bot', tenant: 'acme' });
```

`client.trace.deleteMany()` from `@linnify/agentlens-client` (or the Mastra and LangChain packages' `client`) deletes an agent's traces by `userId`, `tenant` and `environment`, combined, at least one of them. Their observations, scores, reviews and activity go with them, and it resolves to how many traces were deleted.

- It reaches one agent per call, the client's unless `agent` is passed: call it for every agent that may hold the user's traces.
- It cannot be undone. Confirm with the user before running it against production data.
- Call it from the app's account-deletion or tenant-offboarding flow. It is sent once, not retried, and rejects on failure, so the flow has to retry or report it.

## Retention

An agent can keep its traces for a set number of days: older traces, with their reviews, are deleted every night. An owner sets it in the agent's settings with **Edit data retention**, or `save_agent` sets `retentionDays` over MCP. It deletes data, so confirm with the user first.
