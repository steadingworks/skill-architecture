# ADR 000: Positioning relative to MCP and agent skills

## Status
Accepted

## Context

Two prevailing standards exist for extending agent harness functionality:

**Agent skills** are natural language prompt files that extend the agent's
reasoning. They are entirely non-deterministic — the LLM executes everything,
including what would otherwise be deterministic workflows. In practice, D logic
ends up as inline bash commands or local scripts, creating local environment
fragility and making D failures indistinguishable from reasoning failures.

**MCP (Model Context Protocol)** separates tool execution from LLM reasoning
via a structured protocol. MCP supports two transports with meaningfully
different properties:

- **stdio** — the server runs as a local subprocess on the user's machine. No
  network auth needed (OS-level access control is sufficient), but local
  environment fragility remains: language runtimes, package managers,
  permissions, and PATH issues are still the user's problem.

- **SSE** — the server runs remotely over HTTPS. Environment fragility is
  eliminated, but network auth is now required. MCP added an OAuth 2.1 + PKCE
  auth standard in spec version 2025-11-25. The spec is sound, but OAuth 2.1
  is operationally heavy for self-hosted deployments — it requires a full
  authorization server, PKCE flows, and dynamic client registration.

MCP also has no structured place for nD specification — it defines how the
agent calls tools, not when to call them, how to interpret results, or what to
do when things go wrong. That guidance has to live elsewhere.

## Decision

This architecture is not a blanket rejection of MCP. The decision is about
when to use each approach:

**Use an existing SSE MCP server** when one already exists for the domain —
for example, a SaaS product that ships its own MCP integration. In this case,
the D layer is provided and maintained by a third party; the protocol
complexity is already paid. Write a skill prompt as the nD wrapper.

**Build a REST API using this architecture** when you are building the D layer
yourself. For first-party D logic, MCP adds protocol overhead (SSE transport,
OAuth 2.1, spec version tracking, tool schema compliance) with no meaningful
benefit over a plain HTTP service. A FastAPI service with RS256 JWT validation
is simpler to build, deploy, and maintain than an MCP-compliant server.

Empirical research supports this for first-party use: MCP adds 300–800ms
baseline latency over direct REST due to JSON-RPC negotiation, and loads all
tool definitions into context on every invocation, increasing token consumption.
For LLM-dominated workflows, latency adds <5% overhead; token cost is the real
concern, and curated direct API calls are leaner.
(MCP-AgentBench, arxiv:2509.09734; Network and Systems Performance
Characterization of MCP-Enabled LLM Agents, arxiv:2511.07426)

From agent skills: the skill prompt as the explicit, structured nD layer. The
agent knows not just what tools exist but when to use them, how to reason about
results, and what it must not do.

From MCP: the separation of D execution from LLM reasoning, and the principle
that D tools should be defined with a schema (here, `contract.json`).

## Comparison

| | Agent skills | MCP (stdio) | MCP (SSE) | This architecture |
|---|---|---|---|---|
| D/nD separation | ✗ mixed | ✓ | ✓ | ✓ |
| No local D execution | ✗ | ✗ | ✓ | ✓ |
| Auth standard | n/a | not needed | OAuth 2.1 | RS256 JWT |
| Auth complexity | n/a | none | high | low |
| nD specification | ✓ (whole skill) | ✗ | ✗ | ✓ (skill prompt) |
| Runtime portability | Agent Skills-compatible | client-dependent | client-dependent | curl + bash |
| Build cost (first-party) | low | medium | high | medium |
| Token efficiency | n/a | low (all tools in context) | low (all tools in context) | high (curated endpoints) |

## Consequences

- For third-party integrations where an SSE MCP server already exists, use it
  and write a skill prompt wrapper — don't rebuild what's already provided
- For first-party D logic, REST + JWT is simpler to build and operate than an
  MCP-compliant SSE server
- The RS256 JWT auth model is simpler than OAuth 2.1 but has a known
  limitation: it is single-credential, not multi-user. All agents sharing the
  infrastructure share one master API key — there is no per-user identity or
  per-user access control. This is acceptable for solo homelab use but would
  need rethinking for shared deployments
- The skill prompt format follows the Agent Skills open standard, supported
  across Claude Code, GitHub Copilot, Cursor, and VS Code. Runtime-specific
  frontmatter extensions (such as `allowed-tools` in Claude Code) are ignored
  by other runtimes

**Meaningful shortcoming: you need to host the API.** A plain agent skill is a
prompt file. An MCP stdio server runs as a local process. This architecture
requires a deployed HTTP service with a stable hostname, HTTPS, and ongoing
uptime and maintenance. That is a real operational burden that the alternatives avoid: local execution
is fragile and environment-dependent but cheap to run; network-hosted D logic
is reliable and portable but requires infrastructure.
