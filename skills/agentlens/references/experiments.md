# Experiments

An experiment is one run of the project's own dataset against a change to the agent: a new model, effort, instructions or skill, edited in the repository. A local script runs every case through the agent and records it as an **item** of the experiment, with its input, context, expected answer, trace and scores. AgentLens then shows the run's aggregates and compares it with the run before.

The dataset, the script and the grading belong to the project. AgentLens stores the runs. Experiments are created over MCP only, and their tools need the Builder role.

## Recording a run

The script takes an experiment id and wraps each case:

```ts
import { AgentLensClient } from '@linnify/agentlens-client';
import { createTraceId } from '@linnify/agentlens-mastra';

const client = new AgentLensClient({ agent: 'support_bot' });
const experiment = client.experiment(process.env.AGENTLENS_EXPERIMENT_ID!);

for (const testCase of dataset) {
  const traceId = createTraceId();
  const item = experiment.item({
    key: testCase.id,
    input: testCase.question,
    context: { plan: testCase.plan },
    expected: testCase.answer,
    traceId,
  });
  try {
    const result = await agent.generate(testCase.question, {
      tracingOptions: { traceId, metadata: { ...experiment.metadata } },
    });
    item.score({ name: 'Case passed', value: grade(testCase, result) });
  } catch (error) {
    item.fail(error);
  }
}

await client.shutdown(); // sends the items, then the scores
```

| | |
| --- | --- |
| `key` | Identifies the case across experiments, so the same case pairs up in a comparison. Use the dataset row's id. It defaults to a hash of the input |
| `trial` | An integer from 1, default 1. Repeat a case with trials 1, 2, 3 to see how often it passes |
| `input` | Required, any JSON value |
| `context` | An object of key/value pairs the case ran with: tenant, plan, the signed-in customer |
| `expected` | What a good answer is, any JSON value |
| `traceId` | The trace this run will have. When the framework picks it, leave it out and call `item.setTrace(traceId)` after the run |

- `item.fail(error)` records a run that threw. `item.score()` takes the same values as `score.create()` ([Scores](scores.md)) and needs the item's trace.
- Give every case one BOOLEAN score that says whether it passed. A comparison counts a case as fixed or regressed only when a BOOLEAN score flips or a run starts or stops failing.
- The Mastra and LangChain packages' `client` has the same `experiment()`.

The run's trace must carry the experiment's id, or it shows up among the live traces:

| Integration | How the trace gets the experiment id | The trace id |
| --- | --- | --- |
| Mastra | `tracingOptions.metadata: { ...experiment.metadata }` | `createTraceId()` passed as `tracingOptions.traceId`, or `result.traceId` |
| LangChain | Run `metadata: { ...experiment.metadata }`, or the handler's `experimentId` option | Pass your own `runId` (a UUID); the trace is `traceIdFromRunId(runId)` |
| OpenTelemetry | Run it inside `withExperiment(experimentId, () => …)` | `rootSpan.spanContext().traceId` |
| pi | `AGENTLENS_EXPERIMENT_ID` in pi's environment | `AGENTLENS_TRACE_ID`, 32 hex characters, set before spawning pi |

## Trying a change

1. **Baseline.** `list_experiments` for the agent. Use the latest run of the unchanged agent as the baseline. If there is none, or the dataset changed since, run one first: step 3 with nothing changed.
2. **Make the change** in the repository.
3. **Create the experiment.** `create_experiment` with a `name` that says what changed ("effort medium"), a `description`, and `metadata` with the values that define the run (`{ "model": "…", "effort": "medium", "instructions": "<commit or hash>" }`). It returns the id.
4. **Run the project's script with that id** and wait for it to finish.
5. **Read it.** `get_experiment` for the pass rate, each score's aggregate, failed runs, latency, tokens and cost.
6. **Compare.** `compare_experiments` with `baseline` and `candidate`: the cases that were fixed, regressed, mixed (one score better, another worse) or changed, and the score, latency and cost differences. For each regressed case, `list_experiment_items` with its `key` gives the trace id, and `get_trace` shows why.
7. **Report** the change, the pass rate before and after, the fixed and regressed cases with their trace ids, and the cost and latency difference. A change that fixes three cases and breaks two is the user's call.

A model does not answer the same way twice. Before calling a case fixed or regressed on one run, repeat it with several trials and compare how often it passes.

`delete_experiment` removes an experiment with its items and traces: only when the user asks.

## Check it in AgentLens

After a run, `get_experiment` shows as many items as the dataset has cases (times the trials), each with a trace; `list_experiment_items` with `failed: true` lists the runs that threw. `list_traces` for the agent shows none of the run's traces. An item AgentLens refuses (an experiment the organization doesn't have, a trace that already belongs to another item) is logged as `[AgentLens] Experiment item "<key>" (trial <n>) of experiment <id> was rejected: <reason>`.
