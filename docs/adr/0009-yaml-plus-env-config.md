# ADR-0009: YAML for Structure, Environment Variables for Secrets

**Status:** Accepted

## Context

zero-debt has two categories of configuration:

- **Structural** — phase enablement, model roles, thresholds, paths, templates. Should be versionable, reviewable, diffable.
- **Sensitive** — GitHub tokens, LLM API keys, Postgres passwords. Must never hit disk in a committed file, never appear in logs, never be in a container image.

Options considered:

1. **Pure env vars** — works for secrets, painful for structure (nested keys become `ZERO_DEBT__PHASES__SONAR_FIX__MAX_ISSUES_PER_RUN=50` — unreadable, order-sensitive, easy to typo).
2. **Pure YAML** — readable structure, but invites committing secrets "just for dev," then leaking them when someone force-pushes.
3. **Hybrid — YAML for structure, env for secrets.** Standard in containerized apps. Pydantic `BaseSettings` handles both cleanly.

## Decision

Configuration resolves from **two layered sources**:

1. **YAML file** at `$ZERO_DEBT_CONFIG` (default `/app/config/zero-debt.yaml`) — all structural configuration. Committed to the repo.
2. **Environment variables** — all secrets, plus any runtime overrides. Never committed.

Resolution is handled by a Pydantic v2 `BaseSettings` subclass. Env vars take precedence over YAML for overlapping keys. Secrets are **forbidden** from appearing in YAML — a startup check rejects any YAML file containing keys matching the secret-name denylist (`*_TOKEN`, `*_KEY`, `*_SECRET`, `*_PASSWORD`).

Structlog's redactor additionally scrubs these key patterns from log output.

## Consequences

**Positive**

- Structural config is human-readable and reviewable in PRs.
- Secrets live where platforms (Kubernetes, AWS SSM, 1Password injection, etc.) can manage them idiomatically.
- Startup check prevents "accidentally committed secret in YAML" as a failure mode.
- Log redaction catches the secondary leak path.

**Negative**

- Two sources of truth mean slightly more cognitive load when debugging "where did this value come from."
- Pydantic `BaseSettings` adds a small indirection layer — mitigated by centralizing it in `settings.py`.

**Discipline required**

- Never add a secret-shaped key to `zero-debt.yaml` "just for dev." Use a `.env` file (gitignored) with `docker compose --env-file` instead.
- The secret-name denylist in the startup check must stay in sync with the structlog redactor; both reference a shared constants module.
