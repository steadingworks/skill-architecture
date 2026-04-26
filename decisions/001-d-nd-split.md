# ADR 001: Separate deterministic and non-deterministic skill logic

## Status
Accepted

## Context

Agent skills as typically implemented are monolithic: the skill prompt contains
both the reasoning instructions and the execution logic (bash commands, API
calls, data transformations). This creates fragility — D logic running through
the LLM is slower, less reliable, harder to test, and susceptible to local
environment issues (venvs, PATH, permissions, missing tools).

## Decision

Skill prompts handle only nD reasoning. All D workflows are implemented as
pre-written code behind HTTP APIs. The skill prompt tells the agent which API
endpoints to call; the agent executes them via curl through bash.

## Consequences

- D logic is independently testable without an LLM
- D errors surface as HTTP status codes, not as LLM hallucinations
- Local environment fragility is eliminated — deps live in the API container
- Skills become portable: any agent runtime with bash and curl can use them
- Adds operational overhead: D-workflow APIs must be deployed and maintained

## Alternatives considered

**MCP servers**: functionally equivalent but adds protocol complexity, spec
churn, and client dependency. Plain HTTP REST is simpler to deploy, easier to
curl-test, and not tied to a changing spec.

**CLI tools as skill dependencies**: the skill can install or depend on a local
CLI. Rejected — creates the same local environment fragility the split is
designed to avoid.
