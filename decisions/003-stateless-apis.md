# ADR 003: D-workflow APIs must be stateless

## Status
Accepted

## Context

D-workflow APIs are containerised services that may be scheduled across
multiple nodes. Persistent state (volumes, local files) creates node affinity,
complicates scheduling, and breaks horizontal scaling.

## Decision

D-workflow APIs are pure HTTP services: input → output, no volumes, no local
filesystem access beyond injected secrets, no in-process state between requests.
All data needed to fulfil a request must be provided in the request itself.

## Consequences

- No placement constraints needed — services can run on any node
- Stateless services are trivially replaceable: restart, reschedule, or scale
  without data migration
- Secrets are the only external dependency, injected at runtime by the
  container orchestrator
- No cache layer in D-workflow APIs — if caching is needed it is a separate
  infrastructure concern

## Alternatives considered

**Stateful APIs with volumes**: rejected — creates node affinity, complicates
scheduling, and the benefits (caching, persistent connections) are achievable
other ways.
