# Integration

Install the package for the agent's framework, give it the agent's slug, set the trace fields that apply, then run the agent and check the trace in AgentLens. If the project already sends traces to another tool or has its own OpenTelemetry setup, keep it: AgentLens runs next to it.

| Agent built with | Package |
| --- | --- |
| Mastra | `@linnify/agentlens-mastra` |
| LangChain or LangGraph: `createAgent`, `StateGraph`, plain models, chains, tools | `@linnify/agentlens-langchain` |
| pi | `@linnify/agentlens-pi` |
| Any other Node.js code | `@linnify/agentlens-otel` |

## Mastra

```ts
import { AgentLensExporter } from '@linnify/agentlens-mastra';

export const agentLensExporter = new AgentLensExporter({ agent: 'support_bot' });

new Mastra({
  observability: new Observability({
    configs: { default: { serviceName: 'support-bot', exporters: [agentLensExporter] } },
  }),
});
```

- The Mastra thread becomes the trace's session and the Mastra resource its user.
- Per run, `tracingOptions: { metadata: { tenant, userId, sessionId, environment } }`. For a tenant kept on the request context, add `requestContextKeys: ['tenant']` to the config.
- `agentlensAgent` in the same metadata sends that run to another agent.
- Mastra scorer results and `addFeedback()` calls arrive as scores on their trace.

## LangChain and LangGraph

```ts
import { AgentLensCallbackHandler } from '@linnify/agentlens-langchain';

export const agentLens = new AgentLensCallbackHandler({ agent: 'support_bot' });

await agent.invoke(input, {
  callbacks: [agentLens],
  configurable: { thread_id: conversationId },
  metadata: { userId, tenant },
});
```

- `thread_id` becomes the session. Run `metadata` sets `userId`, `tenant`, `sessionId`, `environment` and `agentlensAgent`, and wins over the handler's options.
- The `mask` option runs on every input, output and metadata value before it is sent: use it to strip secrets from graph state.
- When the app does its own steps around the agent, or a tool calls a model with its own callbacks, start the trace yourself: `const root = agentLens.startObservation('support', { input, userId, sessionId }, { asType: 'agent' })`, run the agent inside `root.run(() => …)`, and call `root.end({ output })` last.
- Scripts, serverless functions and tests: `await awaitAllCallbacks()` (from `@langchain/core/callbacks/promises`), then `await agentLens.shutdown()`.

## pi

```bash
pi install npm:@linnify/agentlens-pi
export AGENTLENS_API_KEY=... AGENTLENS_AGENT=builder
```

Each prompt becomes a trace and pi's session its session. `AGENTLENS_USER_ID`, `AGENTLENS_TENANT`, `AGENTLENS_ENVIRONMENT`, `AGENTLENS_SESSION_ID` and `AGENTLENS_METADATA` (a JSON object) set the rest. An app that spawns pi and wants to score the result sets `AGENTLENS_TRACE_ID` to 32 hex characters first.

## OpenTelemetry

```ts
import { AgentLensSpanProcessor } from '@linnify/agentlens-otel';
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node';

const provider = new NodeTracerProvider({
  spanProcessors: [new AgentLensSpanProcessor({ agent: 'support_bot' })],
});
provider.register();
```

Every span the provider starts is sent. AgentLens reads only the `agentlens.*` attributes, so spans from auto-instrumentation (HTTP servers, the Vercel AI SDK) arrive without input, output, model or usage. Set them with `AGENTLENS_ATTRIBUTES`, JSON values as strings:

| Attribute | On | Value |
| --- | --- | --- |
| `OBSERVATION_TYPE` | every span | `'agent'`, `'generation'`, `'tool'` or `'span'` |
| `OBSERVATION_INPUT`, `OBSERVATION_OUTPUT` | every span | JSON |
| `OBSERVATION_MODEL` | generations | the model id, as the provider reports it |
| `OBSERVATION_MODEL_PARAMETERS` | generations | JSON, e.g. `{ "temperature": 0.2 }` |
| `OBSERVATION_USAGE` | generations | `{ "input", "output", "total", "cache_read", "cache_write" }` tokens; `input` includes the cached ones |
| `OBSERVATION_COST` | generations | `{ "input", "output", "total" }` in USD. Leave it out and AgentLens prices known models from the usage |
| `OBSERVATION_METADATA` | any span | a JSON object |
| `TRACE_USER_ID`, `TRACE_SESSION_ID`, `TRACE_TENANT`, `TRACE_ENVIRONMENT` | the root span | strings |

- A span ended with `SpanStatusCode.ERROR` marks the trace as failed, with its message.
- To send a trace to another agent, start its root span inside `withAgent('billing_bot', () => tracer.startActiveSpan(…))`. A root started outside it, such as an HTTP server span, keeps the default agent and the trace splits in two.
- Short-lived processes: `await provider.forceFlush()` before exiting.

## What reviewers read

The trace's input and output come from its root. Make them the user's message and the agent's answer, not the request object or the whole graph state. A list of `{ role, content }` messages, with tool calls, shows as a conversation; any other value shows as JSON. The Mastra and LangChain packages do this already.

## Check it in AgentLens

1. Run the agent once, end to end.
2. Find the trace: `list_traces` with the agent's slug, then `get_trace` (or the `curl` in SKILL.md).
3. Check that:
   - the input is the user's message and the output the agent's answer;
   - each model call is a generation with its model, tokens and cost, each tool call a tool, each sub-agent an agent;
   - session, user, tenant and environment are set wherever the code has them;
   - no secret or unneeded personal data is in any input, output or metadata;
   - a failed run shows as failed.
4. Fix what's off and run it again. Report the trace id.

When no trace arrives, read the app's logs:

| Log | Fix |
| --- | --- |
| `[AgentLens] Missing API key; traces will not be sent` | Load `AGENTLENS_API_KEY` before the SDK is constructed |
| `[AgentLens] Sending N span(s) failed: HTTP 401` | The key is wrong or was deleted |
| `[AgentLens] Span "…" has no agent and was dropped` | Pass `agent` to the SDK, or set it per trace |
| `[AgentLens] N span(s) were rejected: …` | The slug doesn't match an agent of the organization |
| Nothing | `AGENTLENS_ENABLED` is off, or the process exited before flushing |
