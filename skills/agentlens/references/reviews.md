# Reviews

A review is one expert's feedback on one trace. The expert comments right on the output: each comment thread is anchored to the part of the trace it is about, with the quoted text, or is general to the trace, and is resolved or not. A review moves through three statuses:

| Status | Means |
| --- | --- |
| `OPEN` | Waiting on its assignee, or on an owner to assign it |
| `REVIEWED` | Graded, waiting for an owner to decide |
| `CLOSED` | Decided: **Approved** (a good answer, added to the dataset), **Needs improvement** (a wrong answer, added to the dataset to fix) or **Drop** (nothing to keep) |

The MCP server reads reviews. Assigning, resolving, deciding and closing stay with people in AgentLens.

## Fixing the agent from its reviews

1. **Find the agent.** `list_agents` for its slug, and the code that implements it in the workspace.
2. **Read the feedback.** `list_reviews` for the period the user names, or since the agent last changed. Start with the reviews closed as Needs improvement, then the open and reviewed ones with comments. `get_review` for each one's threads.
3. **Read the traces behind them.** `get_trace` for the steps the comments point at. `get_observation` when you need the exact prompt a model was sent or what a tool returned. `list_traces` with `hasError: true` adds failures nobody reviewed yet.
4. **Group by cause.** Six comments saying "quotes last year's prices" are one problem. Find the step where each problem starts (the instructions, the context the agent retrieved, a tool's result, the model's reasoning, the formatting) by reading the steps, not by guessing from the comment. Show the user one table and wait for their go-ahead:

   | Problem | Reviews and traces | Where it starts | Proposed fix |
   | --- | --- | --- | --- |

   A comment asking for a different policy or tone is the user's call, not a bug: list it separately.
5. **Fix it.** Change the prompt, tool or code at the step where the problem starts. Run the agent again on the input of each trace involved, and compare its answers with what the reviewers asked for.
6. **Report** each problem with the trace and review ids it covers, so the reviewers can check the new answers and close their reviews.
