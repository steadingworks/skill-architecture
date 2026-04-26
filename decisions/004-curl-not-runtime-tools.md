# ADR 004: Agent calls D-workflow APIs via curl, not runtime-specific tools

## Status
Accepted

## Context

The agent needs a mechanism to call D-workflow APIs. Agent runtimes vary in
what tools they expose — some have a `web_fetch` tool, some have `http_request`,
some have neither. Depending on a runtime-specific tool creates a portability
constraint.

## Decision

Skill prompts instruct the agent to call D-workflow APIs using `curl` via bash.
`curl` is universally available, well understood by LLMs, and has no runtime
dependency. The `allowed-tools` frontmatter in skill prompts includes
`Bash(curl:*)`.

## Consequences

- Skills work in any agent runtime that has bash access
- curl command examples in the skill prompt serve as both documentation and
  executable instructions
- The agent does not need a special HTTP client tool
- Auth header (`-H "Authorization: Bearer $TOKEN"`) adds one flag per call —
  acceptable overhead

## Alternatives considered

**Runtime web_fetch tool**: rejected — not universally available across agent
runtimes; creates a runtime dependency in the skill.

**MCP tool_call protocol**: rejected — see ADR 001. Adds protocol complexity
without meaningful benefit over curl for this use case.
