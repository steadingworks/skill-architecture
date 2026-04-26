# ADR 002: Shared stateless JWT auth service with asymmetric signing

## Status
Accepted

## Context

D-workflow APIs need to be callable only by authorised agents — not open to
arbitrary requests. The auth mechanism must work across multiple independent API
services without coupling them together.

## Decision

A single shared `skill-auth` service issues RS256 JWTs. D-workflow APIs
validate tokens locally using the public key — no runtime call to the auth
service per request. The auth service has no knowledge of which API services
exist. Adding a new D-workflow API requires no changes to the auth service.

Secrets:
- Private key: Docker Secret, only in `skill-auth`
- Public key: Docker Secret, deployed to each D-workflow API
- Master API key: Docker Secret in `skill-auth` + credential file at
  `~/.config/homelab/skill-apis.api` on the agent host

Tokens: RS256, 1-hour TTL. Agent acquires a token at session start and
re-acquires on 401.

## Consequences

- No shared application secrets between services — compromise of one API
  service does not expose auth credentials
- Auth service is stateless — secrets re-read from disk on each `/token`
  request, so rotation takes effect without restart
- Blast radius of a compromised API service is limited to that service's
  capabilities
- All skills share one credential file — adding a new skill requires no new
  credentials on the agent host

## Alternatives considered

**Per-service bearer tokens**: simpler but requires managing N tokens on the
agent host, and a single compromised token exposes one service's full API.

**mTLS**: achieves the same decoupling at the TLS layer. Operationally heavier
(internal CA, cert issuance and rotation). Worth considering for future
hardening but JWT is sufficient for current scale.

**Shared static token across all services**: rejected — single point of
compromise, all services coupled through the secret.
