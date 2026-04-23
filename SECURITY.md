# Security Policy

## Reporting a vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Email the security contact at `security@<corp.internal>` (replace with the corporate security alias on first deployment) with:

- A description of the issue and its impact.
- Steps to reproduce.
- The version / commit you tested against.
- Any suggested mitigation.

We will acknowledge receipt within **2 business days** and aim to provide an initial assessment within **5 business days**. We commit to fixing critical vulnerabilities within **30 days** of confirmed report.

## Scope

In scope:

- The zero-debt API, worker, web UI, and any official deployment artifacts (Docker images, Helm chart).
- The interaction between zero-debt and external services (GitHub, Anthropic, SonarQube) where the issue stems from zero-debt's handling.
- Authentication, authorization, secret handling, PAT encryption, default-branch protection.

Out of scope (report directly to the relevant vendor):

- GitHub or Anthropic API vulnerabilities themselves.
- Vulnerabilities in dependencies that are not exploitable in zero-debt.
- Self-inflicted misconfigurations (e.g., setting `allow_default_branch=true` and getting a default-branch write).

## Security expectations for contributors

Every PR must respect these. CI enforces what it can; humans enforce the rest.

### Secrets

- **Never commit a secret.** `gitleaks` runs in CI and pre-commit.
- **Never put a secret in YAML.** App startup rejects YAML files containing keys matching `*_TOKEN|*_KEY|*_SECRET|*_PASSWORD`.
- **Never log a secret.** Use the structlog redactor; never `f"token={token}"` even for debug.
- **Never accept a secret as a URL query parameter** (logs and proxies capture them).

### Code

- **No `shell=True`** in subprocess invocations. Argument arrays only.
- **No path traversal.** All filesystem writes go through `FSTool`, jailed under `workspace_root`.
- **No raw SQL with user input.** Parameterized queries only via SQLAlchemy.
- **No deserializing untrusted data** with `pickle`, `yaml.load` (use `safe_load`), or unsigned cookies.
- **No `eval` / `exec`** on any input, ever.

### Auth & authorization

- **Every API endpoint protected** by `require_auth` unless explicitly public (login, request-access, password-reset).
- **Authorization checks at the resource boundary**, not in middleware. The endpoint that returns a `Run` must check `run.user_id == principal.user_id`.
- **404 not 403** for resources the user shouldn't know exist (avoid enumeration).
- **CSRF token** required on all non-GET endpoints when cookie-authenticated.

### Branch & PR safety

- **Default-branch writes refused** unless `allow_default_branch=true`. Two-layer check (tool + branch-name convention).
- **Workspace operations atomic** — failure leaves the working tree at HEAD.

### Dependencies

- **No new dependency without justification.** See [docs/DEPENDENCY_POLICY.md](docs/DEPENDENCY_POLICY.md).
- **`pip-audit`** and **`npm audit`** run in CI; high-severity findings block merge.
- **`trivy`** scans built images.

### LLM-specific

- **No LLM self-eval as a hard gate.** Validation is structural (AST + lint + compile). See [ADR-0005](docs/adr/0005-structural-patch-validation.md).
- **Prompt injection-resistant prompts.** System layer establishes refusal policies for destructive output (e.g., requests to write to `main`, install dependencies, exec shell).
- **Token budgets enforced** server-side; never trusted from input.

## Disclosure

We follow **coordinated disclosure**. Once a fix is shipped:

1. CVE assigned (when applicable).
2. Security advisory published in the repository.
3. CHANGELOG entry under `## Security` for the release.
4. Reporter credited (or kept anonymous if requested).

## Out-of-band incident response

Operational runbook for live incidents lives in `docs/runbooks/incident_response.md` (added with M9 hardening). Until then, the rough flow is: contain → identify → notify maintainers → patch → publish post-mortem.
