# ADR 006: Positioning relative to MCP and agent skills

## Status
Accepted

## Context

Two prevailing standards exist for extending agent harness functionality:

**Agent skills** (e.g. Claude Code skills) are natural language prompt files
that extend the agent's reasoning. They are entirely non-deterministic — the
LLM executes everything, including what would otherwise be deterministic
workflows. In practice, D logic ends up as inline bash commands or local
scripts, creating local environment fragility and making D failures
indistinguishable from reasoning failures.

**MCP (Model Context Protocol)** separates tool execution from LLM reasoning
via a structured protocol. This is the right instinct, but MCP has three
significant shortcomings in practice:

1. **YOLO auth**: The MCP spec defines no authentication standard. Auth is left
   entirely to the implementer, which in practice means either no auth or
   bespoke per-server solutions with no shared conventions.

2. **No nD specification**: MCP defines how the agent calls tools, not how it
   should reason about when to call them, how to interpret results, or what to
   do when things go wrong. This guidance has to live somewhere — typically in
   a generic system prompt — but there is no structured place for it in the MCP
   model.

3. **Local execution**: The canonical MCP transport is stdio — the server runs
   as a subprocess on the user's machine. This reintroduces the local
   environment fragility that tool separation was supposed to solve: language
   runtimes, package managers, permissions, and PATH issues all remain the
   user's problem.

## Decision

This architecture is a middle ground between agent skills and MCP. It takes
what each gets right and addresses what each gets wrong:

| | Agent skills | MCP | This architecture |
|---|---|---|---|
| D/nD separation | ✗ mixed | ✓ | ✓ |
| Auth standard | n/a | ✗ none | ✓ JWT/RS256 |
| nD specification | ✓ (whole skill) | ✗ | ✓ (skill prompt) |
| No local D execution | ✗ | ✗ (stdio) | ✓ (network-hosted) |
| Runtime portability | limited | client-dependent | ✓ (curl + bash) |

From agent skills: the skill prompt as the explicit, structured nD layer. The
agent knows not just what tools exist but when to use them, how to reason about
results, and what it must not do.

From MCP: the separation of D execution from LLM reasoning, and the principle
that D tools should be defined with a schema (here, `contract.json`).

Beyond both: D-workflow APIs are network-hosted services, not local processes.
Auth is a first-class concern with a shared standard across all skills.

## Consequences

- Skills built on this architecture are portable across agent runtimes that
  support bash and curl — no MCP client required, no local server process
- The skill prompt is both the user-facing documentation and the agent's
  operating instructions — it serves the role that MCP's tool schemas and
  system prompt guidance serve separately
- The auth model means adding a new skill requires no new credentials on the
  agent host — a single credential file covers all skills
- Network-hosted D logic is independently testable, deployable, and
  observable without an LLM in the loop

## Relationship to MCP

This architecture does not preclude MCP. A D-workflow API could be wrapped as
an MCP server for runtimes that support it natively. The `contract.json` schema
is close to MCP's `tools/list` response. If MCP clients become ubiquitous and
MCP adds auth conventions, migrating the transport layer would not require
changing the D-workflow API implementations or the skill prompt reasoning.
