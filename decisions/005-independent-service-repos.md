# ADR 005: Each skill is an independent repository

## Status
Accepted

## Context

Skills and their D-workflow APIs have independent lifecycles — they are written,
deployed, updated, and potentially deprecated separately. A monorepo would
couple their release cycles and create noise (CI runs for unrelated skills on
every change).

## Decision

Each skill is a standalone repository containing:
- `api/` — the D-workflow API service (source, Dockerfile, requirements)
- `skill/SKILL.example.md` — skill prompt template with placeholder URLs
- `skill/contract.json` — machine-readable interface spec
- `compose.example.yaml` — deployment template with placeholder config
- `.github/workflows/publish.yml` — builds and pushes image to GHCR on push to `main`

The `skill-auth` service is a separate repository — it is shared infrastructure,
not part of any individual skill.

## What does NOT go in the repo

- **Actual compose files**: deployed compose files with real hostnames,
  certresolver names, and placement constraints live on the Swarm manager, not
  in git. Only example files with placeholders are committed.
- **Actual skill prompts**: installed skill prompts at
  `~/.claude/skills/<name>/SKILL.md` contain real URLs and are never committed.
  Only example files with placeholders are committed.
- **Deploy scripts**: deployment is `docker stack deploy`. Shell scripts that
  encode infrastructure knowledge (SSH targets, remote paths) belong in
  homelab config, not in application repos.

## Consequences

- Each skill can be versioned, released, and deployed independently
- GitHub Actions publishes a new image on every push to `main`, tagged with
  both `latest` and the git SHA
- Rolling back a skill means `docker service update --image ...<sha>`
- The separation between example files (repo) and real deployed files (local)
  must be maintained — it is easy to accidentally commit real config

## Alternatives considered

**Monorepo**: rejected — couples release cycles, creates CI noise, and doesn't
reflect the independent operational lifecycle of each skill.

**Shared platform / registry**: considered for D-workflow APIs — rejected
because it couples services that should be independent, creates a single point
of failure, and destroys skill portability. See the architecture overview.
