# Scores

A score is a named value on a trace, or on one step of it. AgentLens shows a trace's scores on the trace with their comment and source, filters the trace list by them, and charts each one in the agent's metrics.

## Sending a score

Use the client the integration already has:

| Integration | Client |
| --- | --- |
| Mastra | `agentLensExporter.client` |
| LangChain | `agentLens.client` |
| OpenTelemetry, pi, or none | `new AgentLensClient({ agent: 'support_bot' })` from `@linnify/agentlens-client` |

The Mastra and LangChain clients are `undefined` when AgentLens is off, so call them with `?.`:

```ts
agentLensExporter.client?.score.create({
  id: `${traceId}:user_feedback`,
  traceId,
  name: 'user_feedback',
  value: false,
  comment: 'Quoted an outdated price',
  source: 'user',
});
```

| Field | |
| --- | --- |
| `traceId` | Required. The trace the score is about |
| `observationId` | The span id of one step, to score that step instead of the whole trace |
| `name` | Required. The same name for the same signal everywhere: metrics group by name and type |
| `value` | `true`/`false`, a number or a string |
| `dataType` | Inferred from `value`. Set `'TEXT'` for free text, or a string is sent as a category |
| `id` | Sending the same id again updates the score instead of adding one: use it when a user can change their vote |
| `agent` | Defaults to the client's. A trace sent to another agent needs that agent here too |
| `comment`, `source`, `metadata`, `timestamp` | `source` (`user`, `judge`, `ci`…) shows as a badge next to the score |

| Value | Type | In AgentLens |
| --- | --- | --- |
| `true` / `false` | BOOLEAN | 👍 / 👎, a positive rate in metrics, a trace filter |
| number | NUMERIC | the number, its average in metrics |
| string | CATEGORICAL | the label, a trace filter |
| string with `dataType: 'TEXT'` | TEXT | the text |

`score.create()` returns at once and throws only when no agent is set. Scores are sent in the background in batches, and retried on rate limits and server errors. Short-lived processes call `await client.flush()` before exiting.

## Getting the trace id

Feedback usually comes in a later request than the answer: the backend returns the trace id with the answer, and the frontend sends it back with the feedback.

| Integration | Trace id |
| --- | --- |
| Mastra | `result.traceId` from `agent.generate()` |
| LangChain | Pass your own `runId` to the run and use `traceIdFromRunId(runId)`; or `root.traceId` of a `startObservation()` root |
| OpenTelemetry | `rootSpan.spanContext().traceId` |
| pi | The `AGENTLENS_TRACE_ID` the app set before spawning pi |

## Check it in AgentLens

Trigger each score, then `get_trace` its trace: the score is there with the name, value, type, comment and source you meant, on the right step. A rejected score is logged as `[AgentLens] Score <id> was rejected: <reason>`.
