# ADR 006: Observability at the D/nD boundary

## Status
Accepted

## Context

The D/nD split creates a diagnostic seam. When a skill call fails, the agent
sees an HTTP status code and a detail string — but has no visibility into what
happened inside the API or why an upstream behaved as it did. Without
deliberate observability, a 502 looks the same whether the upstream is down, the
API has a bug, or the network has a routing problem. D failures are supposed to
be distinguishable from nD failures; that only holds if the D layer produces
enough signal to diagnose.

## Decision

### API: per-request structured logging

D-workflow APIs must log every request at INFO level using structured output
(JSON). Each log entry must include:

- method, path, response status code
- upstream host called (if any), upstream status received
- response time in milliseconds

Errors logged at ERROR level must include enough detail to diagnose the failure
without an LLM — which upstream, what error class, what was attempted. Auth
failures must be logged distinctly: expired token, invalid token, issuer
mismatch, and scope failure are separate cases.

Credentials and tokens must never appear in logs.

### API: health endpoint must verify dependencies

`/health` must confirm that external dependencies are reachable, not just that
the process is running. A health check that always returns 200 provides no
diagnostic value — a passing health check should mean the service is actually
usable.

### Skill prompt: document non-obvious error actions only

Skill prompts must document error actions that go beyond standard HTTP
semantics:

- **Token lifecycle**: 401 → re-acquire token and retry. The agent cannot infer
  this without knowing how auth works.
- **Skill-specific fallbacks**: paths that only make sense given this skill's
  capabilities (e.g. search unavailable → fall back to fetch).
- **4xx errors must not be retried** — they are client errors that won't
  resolve on retry.

502, 504, and other standard codes do not need spelling out.

## Consequences

- D failures are diagnosable from logs without running the agent
- The agent has enough signal to distinguish "transient, retry" from "broken
  config, escalate to user"
- Health endpoints give a reliable signal for deployment verification and
  container orchestrator liveness checks
- Skill prompts stay concise — error handling rules cover only what the agent
  cannot infer, not the full HTTP status space
