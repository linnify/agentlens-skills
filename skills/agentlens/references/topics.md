# Topics

A topic is a kind of request the agent handles ("Returns and refunds", "Out of scope"). Each has a description and rules that say what belongs to it. A trace is filed under one topic or none; reviews, the Dataset and the trace list are filtered by it.

People create and edit topics in AgentLens. Over MCP you only file traces under the topics that exist, and both tools need the Builder role.

## Filing the traces that have no topic

1. **Read the topics.** `list_topics` returns each topic's id, description and rules, how many traces it has, and how many traces have none.
2. **List the unfiled traces.** `list_traces` with `topic: "none"`, plus the period or filters the user named. Page through them.
3. **Read each one.** `get_trace` for the full input and output. The input usually decides the topic; read the steps only when it doesn't.
4. **Match it against the rules,** not the topic's name. When no topic's rules fit, leave the trace without one: a wrong topic sends it to the wrong reviewers.
5. **File them.** `set_trace_topic` with `traceIds` (up to 100) and one `topic` id. All of them change or none do, so one unknown id fails the call. `topic: "none"` clears a topic.
6. **Report** how many traces went to each topic and how many you left unfiled, with a few examples of the unfiled ones. When they share a kind of request, suggest the topic a person could create for it.

## Check it in AgentLens

`list_topics` again: the count without a topic dropped by what you filed. `get_trace` on one of them shows its topic.
