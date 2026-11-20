# Evaluations

An evaluation runs one version of the agent on its Dataset, the cases experts reviewed, to answer two questions:

| Dataset item | Its question | It passes when |
| --- | --- | --- |
| **Approved**: its output is the expected answer | Did the change break something that worked? | The new answer is the same as the expected one |
| **Open**: a review flagged its output | Did the change fix what the expert flagged? | The new answer fixes what the comments and failed criteria point at |

AgentLens does not run the agent. The project's **harness** does: a service or script that has the agent's code and the configuration of each version. It claims the evaluation, runs every item, reports each result and may decide it. A result it leaves undecided is decided by a person, in AgentLens or through you. Evaluation tools need the Builder role; deciding a result and moving a Dataset item need the Owner role.

## Building the harness

Use the client from `@linnify/agentlens-client` (or the Mastra and LangChain packages' `client`):

```ts
import { AgentLensClient, EvaluationTakenError, parseEvaluationWebhook } from '@linnify/agentlens-client';

const client = new AgentLensClient();

export const runEvaluation = async (evaluationId: string) => {
  let run;
  try {
    run = await client.evaluation(evaluationId).start();
  } catch (error) {
    if (error instanceof EvaluationTakenError) return; // another harness has it, or it already ran
    throw error;
  }

  const version = loadVersion(run.version);
  if (!version) {
    await run.fail(`No configuration for ${run.version}`);
    return;
  }

  for (const item of run.items) {
    const result = run.result(item);
    try {
      const answer = await version.agent.generate(toMessages(item), {
        tracingOptions: { traceId: result.traceId, metadata: { ...run.metadata } },
      });
      if (item.expected) {
        const same = await sameAnswer(item.expected.outputText, answer.text);
        result.score({ name: 'same_answer', value: same.verdict, comment: same.reason });
        result.decide(same.verdict ? 'PASSED' : 'FAILED');
      } else {
        const fixed = await feedbackAddressed(item.feedback, answer.text);
        result.score({ name: 'feedback_addressed', value: fixed.verdict, comment: fixed.reason });
      }
    } catch (error) {
      result.fail(error);
    }
  }

  await run.finish(); // sends the queued results and scores, then ends the run
};

app.post('/agentlens/evaluations', async (req, res) => {
  const { evaluationId } = parseEvaluationWebhook(req.body);
  res.sendStatus(202);
  void runEvaluation(evaluationId);
});
```

- **The webhook is an optional notice.** When the agent has a harness URL, AgentLens posts `{ type: 'evaluation.created', evaluationId, agent, version, items }` to it once. Answer `2xx` at once and run in the background. Its answer never changes the evaluation: with no URL, or when the post fails, the evaluation stays pending and the harness is started by hand with the id, so give the project a command for that (`pnpm run evaluate <evaluationId>`).
- **`start()` claims the evaluation** with the organization's API key. There is no webhook secret: a forged notice names an evaluation the claim refuses. One harness gets the claim; another gets `EvaluationTakenError`.
- **A version is configuration in the repository.** `run.version` is the version's name in AgentLens. Map it to a saved copy of the agent's model, effort, instructions and skills (a folder per version, a tag or a commit), never to whatever is checked out. Fail the run with `run.fail()` when the version is unknown.
- **Each item carries what the run needs:** `itemId`, `status` (`APPROVED` or `OPEN`), `topic`, the `context` it ran in (`tenant`, `userId`, `sessionId`, `environment`, `metadata`), the conversation `history` before it, its `input` and `inputText`, and either `expected` (the approved `output`, `outputText` and the `tools` it called) or `feedback` (the reviewed `output`, the review's `threads` and the `failedCriteria`). It is frozen when the evaluation is created.
- **Run the item as the original ran:** the same tenant and user, the history replayed, on a new session, so that the first run's memory does not leak in.
- **The trace must carry the evaluation's id**, or it shows up among the live traces: `{ ...run.metadata }` in Mastra's `tracingOptions.metadata` or LangChain's run `metadata`, `withEvaluation(run.id, () => …)` in OpenTelemetry, `AGENTLENS_EVALUATION_ID` for pi. `run.result(item)` fixes the trace id before the run; `result.setTrace(traceId)` moves it when the framework picks the id.
- **Decide approved items only when the check is reliable.** Compare what matters, not the wording: the same facts, the same decision, the same actions (a ticket opened, a handover). Answers are free text and ids differ on every run, so an exact match fails everything. When the check is unsure, leave the result undecided.
- **Never decide an open item.** AgentLens refuses it: passing an open item lets an owner approve it, which makes that answer the expected one for every later evaluation. Give it a hint score instead.
- **The harness's decision is final.** Nobody can overturn it in AgentLens; a wrong one is answered with a new evaluation.
- `result.score()` scores the result's trace with `source: 'harness'`. `result.fail(error)` records an item that could not run: it counts as failed, with the error shown. An item the harness never reported fails the same way when the run finishes.
- **Report at least every 15 minutes.** A run that reports nothing for longer fails, and later reports are refused. Reports are batched and retried by the client, so a long item only needs its result created before it runs.

Then set the harness's address on the agent: an owner sets it in the agent's settings with **Edit harness**, or `save_agent` sets `harnessWebhookUrl` over MCP once the user confirms (`null` removes it).

## Running an evaluation

1. **Save the version** in the repository, the way the project's harness reads it.
2. **Create it in AgentLens** when it isn't in `list_versions`: `create_version` with its `name` and a `note` on what changed.
3. **Start it.** `start_evaluation` with the `version` name runs every Approved and Open item; `items` limits it to the Dataset item ids you list. It returns the evaluation's id and `webhookError` when the notice did not reach the harness; start the harness by hand with the id then.
4. **Follow it.** `get_evaluation` with the `id`: `status` goes from `PENDING` (waiting for a harness) to `IN_PROGRESS`, then `DONE` or `FAILED`; `reported` counts the items reported so far and `error` says why a run failed.
5. **Read the results.** The counts of passed, failed and undecided for approved and open items. A failed approved item is a regression: `get_evaluation_result` with the `evaluation` and the `item` shows the input, the expected answer, the new answer and the harness's scores with their reasons.

## Judging what the harness left undecided

1. `get_evaluation` with `result: "UNDECIDED"`.
2. For each one, `get_evaluation_result`: `howToJudge`, the input and history, the new run's output and scores, and the expected answer (approved item) or the original output with the review's threads and failed criteria (open item). `get_trace` on `evalTraceId` when the answer alone doesn't settle it, such as whether a tool was called.
3. Show the user one table and wait for their go-ahead: the item, your verdict, and the reason in one sentence, quoting the answer.
4. `decide_evaluation_result` with `outcome: "PASSED"` or `"FAILED"` for the ones they agree with. A decision cannot be changed. For a result that needs the expert's eye, `send_evaluation_result_to_expert` with the expert's email and a note.
5. An open item that passed can become an expected answer: `approve_dataset_item` with its id, once the user confirms. It is refused for an open item that has not passed an evaluation.

## Improving the agent from its Dataset

1. `list_dataset_items` with `status: "OPEN"`, and `get_dataset_item` for each one's input, original output and review feedback.
2. Group them by cause and propose the fixes, as in [Reviews](reviews.md). Change the model, effort, instructions or skills in the repository.
3. Try the change on the project's own dataset first, when it has one: [Experiments](experiments.md).
4. Save a new version, run an evaluation of it, and judge the undecided results.
5. Report the open items that now pass, and every approved item that failed: a regression is the price of the change, and whether to pay it is the user's call.

## Check it in AgentLens

- After the harness's first run, `get_evaluation` shows `DONE` with `reported` equal to the number of items, and every result has an `evalTraceId`.
- `list_traces` for the agent shows none of the evaluation's traces.
- A result AgentLens refuses is logged by the client as `[AgentLens] The evaluation result of item <id> was rejected: <reason>`.
