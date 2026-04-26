# Skill Architecture

Architecture documentation and playbook for building agent skills using the
D/nD (deterministic / non-deterministic) pattern.

## The core idea

Agent skills mix two kinds of work:

- **Non-deterministic (nD)** — reasoning, judgment, synthesis. Only the LLM
  can do this. Lives in the skill prompt.
- **Deterministic (D)** — fetching, transforming, calling APIs, running
  commands. Pre-written code can do this reliably, testably, without an LLM.

The pattern: **skill prompts are thin**. They handle nD reasoning and direct the
agent to call D-workflow APIs for everything else. D logic runs as a stateless
HTTP service, isolated from the agent's local environment.

## Why this matters

Without this split, D logic ends up in the skill prompt as bash commands or
inline scripts. This means:

- Local environment fragility (venvs, PATH, permissions)
- No testability without running the full agent
- D errors look like nD errors — hard to debug
- Logic that should be deterministic gets re-executed by an LLM

## Reference implementations

- [`skill-auth`](https://github.com/steadingworks/skill-auth) — shared RS256
  JWT auth service. Every D-workflow API validates tokens from here.
- [`skill-web-fetch`](https://github.com/steadingworks/skill-web-fetch) — web
  fetch and search. The canonical reference skill implementation.

## Documents

- [Playbook](playbook.md) — how to build a new skill end-to-end
- [Decisions](decisions/) — architecture decision records (ADRs)

## Status

Living document. Implementations may periodically drift from this spec as the
pattern evolves. When they do, this repo is updated first.
