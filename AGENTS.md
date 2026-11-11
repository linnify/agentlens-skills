# AgentLens Skills

The `agentlens` skill teaches a coding agent to connect an AI agent to AgentLens. `SKILL.md` holds what every task needs (the concepts, the setup, the environment variables, the MCP server) and links to one section per task under `references/`.

## What the skill must match

Everything the skill states comes from two repositories, and must stay true to what is released:

| Repository | What the skill takes from it |
| --- | --- |
| [linnify/agentlens-sdk](https://github.com/linnify/agentlens-sdk) | Package names, options, environment variables, span attributes, log messages, code samples |
| [linnify/linnify-agentlens](https://github.com/linnify/linnify-agentlens) | The MCP server's URL and tools, the ingestion API, screen and setting names, review statuses and decisions, how scores are shown |

When a release of either renames, adds or removes something the skill mentions, update the skill.

## Changing the skill

- A new kind of task gets its own section in `references/`, with one line in `SKILL.md`'s **Sections** list. Anything else extends an existing section.
- Write for an agent in the middle of the task: what to do, the values to use, how to check it worked. Leave out what it can read from the code in front of it.
- Keep code samples minimal and copy them from the SDK's released READMEs and source.
- `allowed-tools` lists only tools that read. Never `save_agent`, or anything that writes or deletes.

## Releasing

`main` is the only branch: pull requests are squash-merged into it with a conventional-commit title, and users install from it.

Bump `version` in `.claude-plugin/plugin.json` in the same pull request as any change under `skills/`: patch for a fix, minor for a new section or capability, major for a removed or renamed skill.
