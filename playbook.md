# Skill Playbook

Step-by-step guide for building a new skill using the D/nD pattern.

## Before you start

Define the D/nD boundary for your skill. Ask:

- What requires judgment, context, or synthesis? → nD, goes in the skill prompt
- What is a pure function given known inputs? → D, goes in the API

If you find yourself putting conditionals, retries, or result interpretation in
the API, it's nD — move it to the prompt. If you find yourself putting HTTP
calls or data transformation in the prompt, it's D — move it to the API.

---

## 1. Create the D-workflow API

Use [`skill-web-fetch/api/`](https://github.com/steadingworks/skill-web-fetch)
as the reference structure.

### Structure

```
my-skill/
└── api/
    ├── src/
    │   ├── main.py          # FastAPI app wiring
    │   ├── config.py        # Pydantic settings (MY_SKILL_ prefix)
    │   ├── auth.py          # JWT validation — copy verbatim from reference
    │   └── routes/
    │       ├── health.py    # GET /health — no auth, copy verbatim
    │       └── *.py         # Your D-workflow endpoints
    ├── Dockerfile           # Two-stage build — copy from reference, change port
    └── requirements.txt
```

### Rules for D-workflow endpoints

- **Stateless**: no volumes, no local filesystem, no in-process state
- **Pure**: same input always produces same output (given external services behave)
- **Error mapping**: timeout→504, upstream HTTP error→502, bad input→400
- **Content limits**: cap response sizes before returning to the agent
- **Auth**: every endpoint (except `/health`) requires a valid JWT via
  `Depends(require_jwt)` — copy `auth.py` verbatim from the reference

### config.py

```python
class Settings(BaseSettings):
    model_config = {"env_prefix": "MY_SKILL_"}

    jwt_public_key_path: str   # path to public key file
    jwt_issuer: str = "skill-auth"
    log_level: str = "INFO"
    # ... your settings
```

### Dockerfile

Copy the reference Dockerfile. Change:
- The port (`EXPOSE` and `--port`)
- Nothing else — the two-stage build, non-root user, and healthcheck are correct

---

## 2. Write the contract

`skill/contract.json` is the machine-readable interface spec. Every endpoint,
parameter, response shape, and error code goes here. This is the source of
truth — the skill prompt and API implementation are both derived from it.

See [`skill-web-fetch/skill/contract.json`](https://github.com/steadingworks/skill-web-fetch/blob/main/skill/contract.json)
for the full schema.

Minimum required fields:

```json
{
  "schema": "skill-contract/v1",
  "name": "my-skill",
  "version": "1.0.0",
  "auth": {
    "type": "bearer-jwt",
    "algorithm": "RS256",
    "token_endpoint": "https://<your-auth-host>/token",
    "token_request_header": "X-API-Key",
    "credential_file": "~/.config/homelab/skill-apis.api",
    "credential_key": "key",
    "token_ttl_seconds": 3600
  },
  "base_url": "https://<your-api-host>",
  "side_effects": "none",
  "endpoints": [...]
}
```

Set `side_effects` to `"none"` if the API only reads, or describe the mutation
(e.g. `"writes to library database"`). The agent uses this to decide whether
retrying a failed request is safe.

---

## 3. Write the skill prompt

`skill/SKILL.example.md` is committed to the repo with placeholder URLs.
The installed copy at `~/.claude/skills/<skill-name>/SKILL.md` has real URLs
and is never committed.

### Structure

```markdown
---
name: my-skill
description: <one line, describes what the skill does and when to trigger it>
allowed-tools: Bash(curl:*), Bash(grep:*), Bash(python3:*)
user-invocable: true
---

# my-skill

<One paragraph: what this skill does and what it does NOT do>

## Setup: Acquire a Token

Run once per session. Re-run if any API call returns 401.

[token acquisition curl command]

Never print or echo $MY_SKILL_TOKEN.

## Endpoints

[One section per endpoint with copy-paste curl examples]

## How to approach tasks

[Decision rules: when to call which endpoint, how to handle errors]

## Must NOT

[Hard constraints on what the agent must not do]
```

### What makes a good skill prompt

- The agent should be able to use the skill having read only the prompt — no
  external docs needed
- Every endpoint gets a copy-paste curl example, not just a description
- Error handling rules are explicit: "on 401, do X; on 503, do Y"
- "Must NOT" section is short and absolute — only things the agent might
  actually try to do without guidance

---

## 4. GitHub Actions

Copy `.github/workflows/publish.yml` from either reference repo. Change the
image name. The workflow builds and pushes to GHCR on every push to `main`.

---

## 5. Deployment

### Requirements

The D-workflow API is a stateless HTTP service. It can run anywhere that can:

- Pull a container image from GHCR
- Serve HTTPS on a stable hostname reachable by the agent host
- Inject secrets as files or environment variables at runtime
- Restart the container on failure

This includes Docker Swarm, k3s, k8s, Fly.io, Railway, or a plain `docker run`
behind a reverse proxy. The `compose.example.yaml` in each repo is written for
Docker Compose/Swarm — adapt it to your platform.

### Secrets

Your API needs the `skill-auth` public key injected at the path configured by
`MY_SKILL_JWT_PUBLIC_KEY_PATH`. How you deliver it depends on your platform
(Docker Secret, k8s Secret, environment variable pointing to a mounted file,
etc.).

If your skill needs additional secrets (API keys, external service credentials),
inject them the same way and read the paths from environment variables in
`config.py`.

### Environment variables

At minimum:

```
MY_SKILL_JWT_PUBLIC_KEY_PATH=<path to public key file>
MY_SKILL_JWT_ISSUER=<hostname of skill-auth service>
```

### Updates

After a push to `main`, GitHub Actions builds and pushes a new image tagged
with both `latest` and the git SHA. Deploy the SHA-tagged image rather than
`latest` — it guarantees your orchestrator pulls the new image rather than
using a cached layer.

See [`skill-web-fetch`](https://github.com/steadingworks/skill-web-fetch) for
a concrete deployment example using Docker Swarm.

---

## 6. Definition of done

- [ ] D-workflow API builds and health-checks locally (`docker run`)
- [ ] JWT validation rejects invalid/expired tokens (401) and missing tokens (403)
- [ ] All D endpoints return structured errors, not stack traces
- [ ] `contract.json` matches the implemented endpoints exactly
- [ ] `SKILL.example.md` has placeholder URLs, no environment-specific content
- [ ] `compose.example.yaml` has placeholder hostnames and certresolver
- [ ] GitHub Actions publishes multi-arch image on push to `main`
- [ ] Skill installed at `~/.claude/skills/<name>/SKILL.md` with real URLs
- [ ] End-to-end test: agent session uses the skill without credentials
  appearing in the conversation
