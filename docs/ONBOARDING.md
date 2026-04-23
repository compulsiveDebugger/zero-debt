# Developer Onboarding

Day-1 setup for engineers joining zero-debt. Follow top to bottom; expect ~2 hours from clone to first green local run.

If anything in here is wrong or out of date, **fix it in the same PR you discover the issue**. Onboarding rot is the worst kind.

---

## 0. Prerequisites

Install on your host:

| Tool | Version | Notes |
|---|---|---|
| **Docker Engine** | 24.x+ | Includes Compose v2 |
| **Git** | 2.40+ | |
| **Python** | 3.11+ | For running tests outside the container |
| **Node.js** | 20+ | For Angular dev server |
| **uv** | 0.4+ | Faster than pip for dep resolution |
| **gh** (GitHub CLI) | latest | For PR ergonomics |

Optional but recommended: `jq`, `psql`, `redis-cli`, `mongosh`, `make`.

---

## 1. Clone and configure

```bash
git clone <repo-url> zero-debt
cd zero-debt

# Copy the example env; fill in real values
cp .env.example .env
```

`.env` you'll need (minimum to start the stack):

```bash
ANTHROPIC_API_KEY=sk-ant-...        # ask your team lead for the dev key
GITHUB_TOKEN=                        # optional for service-account ops
POSTGRES_PASSWORD=devpw_change_me
MONGO_PASSWORD=devpw_change_me
REDIS_PASSWORD=devpw_change_me
SESSION_SECRET=$(openssl rand -hex 32)
PAT_ENCRYPTION_KEY=$(openssl rand -base64 32)
```

> **Never commit `.env`.** The `.gitignore` covers it; the pre-commit hook double-checks.

---

## 2. Install dev dependencies

### Python (backend + worker)

```bash
uv venv                              # creates .venv
source .venv/bin/activate
uv pip install -e ".[dev]"
pre-commit install                   # installs git hooks
pre-commit install --hook-type commit-msg
```

### Frontend

```bash
cd web
npm install
cd ..
```

---

## 3. Bring up the stack

```bash
docker compose up -d postgres redis mongo
```

Wait ~10 seconds, then run migrations:

```bash
alembic upgrade head
```

Start the API and worker locally (faster iteration than container rebuilds):

```bash
# Terminal 1 — API
uvicorn zero_debt.api.main:app --reload --port 8000

# Terminal 2 — Worker
arq zero_debt.worker.main.WorkerSettings

# Terminal 3 — Frontend dev server
cd web && npm start                  # serves on :4200, proxies /api → :8000
```

Open `http://localhost:4200`. You should see the login screen.

---

## 4. Seed your first user

```bash
ZD_SEED_ADMIN=1 python -m zero_debt.scripts.seed_admin \
  --email you@corp.internal --password ChangeMe123
```

Log in. Connect a test repo (use a personal repo or any public one — PAT with `repo` scope).

---

## 5. Run a Phase 1 fixture

We ship a fixture Sonar report and a known fix-friendly repo for smoke testing.

```bash
# From the UI:
# - Connect repo: https://github.com/<your-fixture-org>/sonar-fix-fixture
# - New Run → Sonar Fix → Upload tests/fixtures/sonar_reports/checkout-svc-50.json
# - Click Enqueue
```

Watch the live progress stream. Within 5–10 minutes you should see a draft PR open on the fixture repo.

---

## 6. Run the test suite

```bash
# Backend unit tests — should be <30s
pytest tests/unit -q

# Backend integration tests — needs Postgres + Redis + Mongo running
pytest tests/integration -q

# Frontend unit tests
cd web && npm test -- --watch=false

# Frontend e2e (Playwright) — boots the stack via docker-compose.test.yml
cd web && npm run e2e
```

Coverage reports land in `htmlcov/` (Python) and `web/coverage/` (Angular).

---

## 7. Editor setup

### VS Code

Recommended extensions:

- **Python** + **Pylance** — strict type checking
- **Ruff** — fast linter / formatter
- **ESLint** + **Prettier**
- **Angular Language Service**
- **Even Better TOML**, **YAML**

`.vscode/settings.json` (commit a `settings.example.json`, gitignore the live one):

```json
{
  "python.analysis.typeCheckingMode": "strict",
  "[python]": { "editor.defaultFormatter": "charliermarsh.ruff" },
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.organizeImports": "explicit" },
  "files.insertFinalNewline": true
}
```

### JetBrains (PyCharm / WebStorm / IDEA)

- Project SDK: Python 3.11+ from `.venv`.
- Enable **EditorConfig support** (bundled).
- Install **Ruff** plugin.
- For Angular: enable **Angular and AngularJS** plugin.

---

## 8. What to read next

In this order:

1. [README.md](../README.md) — product framing
2. [docs/GLOSSARY.md](GLOSSARY.md) — domain vocabulary
3. [docs/ARCHITECTURE.md](ARCHITECTURE.md) — full architecture
4. [docs/PHASE-1-BLUEPRINT.md](PHASE-1-BLUEPRINT.md) — what we're building first
5. [docs/02_implementation_plan.md](02_implementation_plan.md) — the work plan
6. The relevant ADRs in [docs/adr/](adr/) for the area you'll touch
7. The code style guide for your stack: [Python](CODE_STYLE_PYTHON.md) or [Angular](CODE_STYLE_ANGULAR.md)
8. [docs/TESTING.md](TESTING.md) before you write your first test
9. [CONTRIBUTING.md](../CONTRIBUTING.md) before you open your first PR

---

## 9. Common first-day pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `psycopg.OperationalError: connection refused` | Postgres not up yet | `docker compose ps`; wait for `(healthy)` |
| `ANTHROPIC_API_KEY missing` at startup | Forgot to source `.env` | `set -a; source .env; set +a` |
| Pre-commit fails on first commit with mypy errors | Stale type cache | `rm -rf .mypy_cache && pre-commit run mypy --all-files` |
| Angular build fails: `Cannot find module @api-generated/...` | Generated client out of sync | `npm run gen:api` (needs API running) |
| SSE connection closes after ~30s in browser | nginx idle timeout | Use the dev server (`npm start`) which proxies cleanly |
| Worker logs `RedisUnreachable` on startup | Wrong `REDIS_URL` | Check `.env` matches the Compose service name (`redis`) |

---

## 10. Asking for help

- Quick questions → team chat channel `#zero-debt-dev`.
- Code-level confusion → leave a comment on the file in your branch and tag a maintainer.
- "I think the architecture is wrong" → open an RFC issue. We'd rather hear it than have you work around it.
