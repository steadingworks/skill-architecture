# ADR 005: Each skill is an independent repository

## Status
Accepted

## Context

Skills and their D-workflow APIs have independent lifecycles — they are written,
deployed, updated, and potentially deprecated separately. A monorepo would
couple their release cycles and create noise (CI builds for unrelated skills on
every change).

## Decision

Each skill is a standalone repository containing:
- `api/` — the D-workflow API service (source, Dockerfile, dependencies)
- `skill/SKILL.example.md` — skill prompt template with placeholder URLs
- `skill/contract.json` — machine-readable interface spec
- `compose.example.yaml` — deployment template with placeholder config
- `.github/workflows/publish.yml` — builds and pushes image to GHCR on push to `main`

The `skill-auth` service is a separate repository — it is shared infrastructure,
not part of any individual skill.

## What does NOT go in the repo

- **Actual deployment config**: files with real hostnames and environment-specific
  settings live in the deployment environment, not in git. Only example files
  with placeholders are committed.
- **Actual skill prompts**: installed skill prompts contain real URLs and are
  never committed. Only example files with placeholders are committed.
- **Deploy scripts**: deployment is a standard container orchestration operation.
  Scripts that encode environment-specific knowledge (hosts, paths, tooling)
  belong in infrastructure config, not in application repos.

## Consequences

- Each skill can be versioned, released, and deployed independently
- CI publishes a new image on every push to `main`, tagged with both `latest`
  and the git SHA
- Rolling back a skill means updating the service to a previous SHA tag
- The separation between example files (repo) and real deployed files (local)
  must be maintained. Each skill repo includes a `.gitignore` that blocks the
  most likely accidents:
  ```
  skill/SKILL.md
  compose.yaml
  ```
  Before pushing, verify that no files containing real hostnames, tokens, or
  paths are staged.

## Alternatives considered

**Monorepo**: rejected — couples release cycles, creates CI noise, and doesn't
reflect the independent operational lifecycle of each skill.

**Shared platform / registry**: considered for D-workflow APIs — rejected
because it couples services that should be independent, creates a single point
of failure, and destroys skill portability.
