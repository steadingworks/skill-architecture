# ADR 003: D-workflow APIs must be stateless

## Status
Accepted

## Context

D-workflow APIs run on a Docker Swarm cluster. Services can be scheduled on any
node. Persistent state (volumes, local files) would pin services to specific
nodes, break horizontal scaling, and complicate deployment.

## Decision

D-workflow APIs are pure HTTP services: input → output, no volumes, no local
filesystem access beyond mounted secrets, no in-process state between requests.
All data needed to fulfil a request must be provided in the request itself.

## Consequences

- No placement constraints needed — services can run on any node
- Stateless services are trivially replaceable: restart, reschedule, scale
  without data migration
- Secrets are the only external dependency, mounted as tmpfs via Docker Secrets
- No cache layer in D-workflow APIs — if caching is needed, it is a separate
  infrastructure concern

## Alternatives considered

**Stateful APIs with volumes**: rejected — creates node affinity, complicates
Swarm scheduling, and the benefits (caching, persistent connections) are
achievable other ways.
