# zero-debt — Tools Inventory

Complete catalog organized in four tiers:

1. **External services** — things we talk to over the network.
2. **Runtime dependencies** — packages bundled into containers (Python + Node).
3. **Internal tool abstractions** — the `tools/` layer every node calls into.
4. **Developer / operator tools** — what a human needs to build and operate zero-debt.

---

## 1. External Services

| Service | Purpose | Auth | Phases |
|---|---|---|---|
| **GitHub REST API v3** | Clone, branch, commit, push, PR | User PAT stored encrypted (`RepoConnection.pat_secret_ref`) or GitHub OAuth | All |
| **GitHub Webhooks** (optional) | Trigger P3 on PR events | Shared-secret HMAC | P3 |
| **SonarQube Web API** | Fetch issues live in lieu of JSON export | `SONARQUBE_TOKEN` | P1 |
| **Anthropic Messages API** | Primary LLM provider | `ANTHROPIC_API_KEY` | All |
| **OpenAI Chat Completions API** (optional) | Alternate LLM provider; also used for embeddings in v1 | `OPENAI_API_KEY` | All / semantic index |
| **ZTIAP** (post budget approval) | Corporate auth | ZTIAP token introspection | All |
| **OpenTelemetry Collector** (optional) | Trace export | OTLP endpoint | All |
| **Application under test** (P5) | Target for Playwright + visual baselines | User-configured | P5 |

All external-service clients pass through the tool layer (§3). No LangGraph node touches a raw SDK.

---

## 2. Runtime Dependencies

### 2.1 Python (Backend + Worker)

#### Orchestration & State
| Package | Purpose | Version floor |
|---|---|---|
| `langgraph` | StateGraph, subgraphs, checkpointer | `>=0.2.0` |
| `langchain-core` | Message & tool primitives | `>=0.3.0` |

#### Contracts & Settings
| Package | Purpose | Version floor |
|---|---|---|
| `pydantic` | Data contracts for every external payload | `>=2.6` |
| `pydantic-settings` | Env + YAML config loading via `BaseSettings` | `>=2.2` |
| `PyYAML` | YAML parser | `>=6.0` |

#### HTTP & Web Tier
| Package | Purpose | Version floor |
|---|---|---|
| `fastapi` | HTTP API + SSE | `>=0.110` |
| `uvicorn[standard]` | ASGI server | `>=0.29` |
| `httpx` | Async HTTP client | `>=0.27` |
| `sse-starlette` | Server-Sent Events helpers for FastAPI | `>=2.1` |
| `jinja2` | PR body + report templating | `>=3.1` |

#### CLI (for ops and tests)
| Package | Purpose | Version floor |
|---|---|---|
| `typer` | CLI entry point | `>=0.12` |

#### LLM Providers
| Package | Purpose | Version floor |
|---|---|---|
| `anthropic` | Claude SDK | `>=0.40` |
| `openai` | OpenAI SDK (alt provider + embeddings) | `>=1.40` |

#### Git & GitHub
| Package | Purpose | Version floor |
|---|---|---|
| `PyGithub` | REST wrapper for PR / branch / review ops | `>=2.3` |
| `GitPython` | Local git operations | `>=3.1` |
| `unidiff` | Parse / validate unified diffs | `>=0.7` |

#### Queue & Cache
| Package | Purpose | Version floor |
|---|---|---|
| `arq` | Async Redis job queue | `>=0.26` |
| `redis` | Python Redis client (async) | `>=5.0` |

#### Persistence
| Package | Purpose | Version floor |
|---|---|---|
| `psycopg[binary]` | Postgres driver | `>=3.1` |
| `sqlalchemy` | ORM for app data + migrations | `>=2.0` |
| `alembic` | Migrations | `>=1.13` |
| `pgvector` | pgvector Python bindings | `>=0.3` |
| `motor` | Async MongoDB driver | `>=3.4` |

#### Auth
| Package | Purpose | Version floor |
|---|---|---|
| `argon2-cffi` | Argon2id password hashing | `>=23.1` |
| `itsdangerous` | Signed session token generation | `>=2.2` |
| `python-jose[cryptography]` | JWT (for ZTIAP later; optional v1) | `>=3.3` |
| `cryptography` | PAT encryption at rest | `>=42.0` |

#### Language Tooling (Validator)
| Package | Purpose | Version floor |
|---|---|---|
| `tree-sitter` | Breadth AST parser | `>=0.22` |
| `tree-sitter-languages` | Pre-built grammars (Python, Java, Rust, TS, Go, more) | `>=1.10` |

> Per-language native toolchains (javac, tsc, ruff, cargo) invoked via `LanguageTool`. See §4.3 for which toolchains must be installed in the worker image.

#### Resilience & Observability
| Package | Purpose | Version floor |
|---|---|---|
| `tenacity` | Exponential backoff | `>=8.2` |
| `structlog` | Structured logging | `>=24.1` |
| `python-json-logger` | stdlib bridge | `>=2.0` |
| `opentelemetry-api` / `-sdk` | Traces | `>=1.25` |
| `prometheus-client` | `/metrics` endpoint | `>=0.20` |

#### Semantic Indexing (post-P1)
| Package | Purpose | Version floor |
|---|---|---|
| `sentence-transformers` (optional) | Local embeddings if `OPENAI_API_KEY` not used | `>=3.0` |

#### Optional Extras
| Extra | Packages | When |
|---|---|---|
| `playwright` | `playwright>=1.44` | P5 enabled |
| `dev` | `pytest`, `pytest-asyncio`, `ruff`, `mypy`, `respx`, `httpx`, `freezegun` | Local dev |

### 2.2 Node / Angular (Frontend)

| Package | Purpose | Version floor |
|---|---|---|
| `@angular/core`, `/common`, `/forms`, `/router` | Framework | `^17` |
| `@angular/material` | UI primitives | `^17` |
| `rxjs` | Reactive streams (SSE handling) | `^7.8` |
| `@angular/platform-browser` | Runtime | `^17` |
| `tailwindcss` (optional) | Styling utility | `^3.4` |
| `@angular/cli`, `typescript`, `karma`, `jasmine` | Dev tooling | matched to Angular version |

Build output served by **nginx** in a dedicated container.

---

## 3. Internal Tool Abstractions

Stateless classes, async-first, registered in a `ToolRegistry` sourced from `state.config`. **No LangGraph node instantiates an SDK client directly.**

| Tool | File | Responsibility | Used by |
|---|---|---|---|
| `GitHubTool` | `tools/github_tool.py` | Clone, branch, commit/push, PR create/comment/review. Wraps `PyGithub` + `GitPython`. | `clone_or_fetch_repo`, `commit_and_push_branch`, `open_pull_request`, P3, P4 |
| `GitTool` | `tools/git_tool.py` | Low-level git (status, diff, reset, stash). Args-as-array only. | `commit_and_push_branch`, P1 rollback, P4 |
| `FSTool` | `tools/fs_tool.py` | Workspace-scoped read/write. Path-traversal jailed. | Every subgraph |
| `SonarTool` | `tools/sonar_tool.py` | Parse JSON reports; optional live client. | P1 |
| `LanguageTool` | `tools/language_tool.py` | Strategy pattern: AST parse, lint delta, format, optional compile. **Strategies: Java, Python, Rust, TS, Go.** | P1 validator, P2, P4 |
| `PatchTool` | `tools/patch_tool.py` | Apply unified diffs with rollback; validates via `unidiff`. | P1 `apply_patch`, P4 |
| `PlaywrightTool` | `tools/playwright_tool.py` | Scaffold config, fixtures, specs; capture baselines. | P5 |
| `ReportTool` | `tools/report_tool.py` | Jinja2 rendering for PR bodies and summary reports. | `generate_summary_report` |
| `TelemetrySink` | `tools/telemetry_sink.py` | Writes events / LLM interactions / audit to MongoDB. | Every node via middleware |
| `VectorStore` | `vector/store.py` | `upsert`, `query_neighbors`, `delete_repo`. `pgvector` default; `qdrant` future. | Post-P1 phases |
| `LLMGateway` | `llm/gateway.py` | Single abstraction for all LLM calls. Logical model ids. Capability probe. | Every LLM-using subgraph |
| `AuthProvider` | `auth/provider.py` | `login`, `verify_session`, `logout`. `DbAuthProvider` v1; `ZtiapAuthProvider` v2. | API middleware |

### 3.1 Cross-cutting safety contracts

Every tool must:

1. Jail filesystem writes under `config.workspace_root`.
2. Refuse GitHub mutations targeting default / `main` / `master` unless `config.allow_default_branch=true`.
3. Use argument arrays for all subprocess calls (no `shell=True`).
4. Emit a structured log line per op (`run_id`, `tool`, `op`, `duration_ms`, `upstream_request_id`).
5. Never log secrets — rely on the structlog redactor *and* never pass tokens into a log call directly.

---

## 4. Developer & Operator Tools

### 4.1 Required on the host

| Tool | Purpose | Min version |
|---|---|---|
| **Docker Engine** | Container runtime | 24.x |
| **Docker Compose** | Local stack orchestration | v2.23+ |
| **Git** | Cloning zero-debt | 2.40+ |
| **Python** | Only for local (outside-container) backend dev | 3.11+ |
| **Node.js** | Angular build / UI dev | 20+ |
| **npm** or **pnpm** | UI package manager | npm 10+ / pnpm 9+ |
| **uv** or **pip** | Python deps | `uv>=0.4` preferred |
| **make** (optional) | Shorthand for common dev tasks | — |

### 4.2 Recommended

| Tool | Purpose |
|---|---|
| `gh` | GitHub CLI — inspect PRs zero-debt opens |
| `jq` | JSON report inspection |
| `psql`, `redis-cli`, `mongosh` | Direct data store inspection |
| `ruff`, `mypy` | Matches in-container lint/type gate |
| `pre-commit` | Local enforcement |

### 4.3 Container images

| Image | Used by | Size (rough) | Notes |
|---|---|---|---|
| `python:3.11-slim` (or `rockylinux:9-minimal` + py install) | `api` | ~150 MB | Keep slim |
| Same base **+ per-language toolchains** | `worker` | ~800 MB — 1.2 GB | Bundles `cargo`, a JDK, `tsc`, Python tooling. See §4.3.1 |
| `pgvector/pgvector:pg16` | `postgres` | ~280 MB | Postgres with pgvector preloaded |
| `redis:7-alpine` | `redis` | ~40 MB | |
| `mongo:7` | `mongo` | ~720 MB | |
| `nginx:1.27-alpine` (used inside `web-ui` image) | `web-ui` | ~40 MB | Plus the Angular build artifact |
| `mcr.microsoft.com/playwright:v1.44.0-jammy` | `playwright-runner` | ~1.5 GB | Only `profiles: ["phase5"]` |

#### 4.3.1 Worker toolchain matrix

P2 requires language-native tooling inside the worker for high-fidelity validation and test-framework scaffolding.

| Language | Tooling bundled in worker image | Used for |
|---|---|---|
| Python | `python3.11`, `pip`, `ruff`, `mypy`, `pytest` | P1 validator, P2 test gen + coverage delta |
| Java | `openjdk-17`, `maven` or `gradle` | P1 validator (compile), P2 JUnit scaffolding |
| Rust | `rustup` (stable), `cargo`, `clippy`, `rustfmt` | P1 validator (rarely), P2 `cargo test` scaffolding |
| TypeScript | `node`, `tsc`, `eslint` | P1 validator, P2 Jest scaffolding |

> Alternative (deferred): split worker into per-language sidecar runners invoked via a local gRPC or exec socket. Keeps each image slim but multiplies ops surface. Decision to revisit after M5 if the fat-image size becomes a real problem.

### 4.4 CI/CD

| Stage | Tooling |
|---|---|
| Lint (Python) | `ruff check` |
| Type-check | `mypy --strict` on `src/zero_debt` |
| Lint (UI) | `ng lint` |
| Unit tests | `pytest -q tests/unit`, `ng test --watch=false` |
| Integration tests | `pytest -q tests/integration` (needs Postgres + Redis + Mongo via `compose.test.yml`) |
| E2E tests | Playwright against the running stack |
| Container build | `docker buildx build --platform linux/amd64` per Dockerfile |
| Security scan | `trivy image`; `gitleaks` on commits |
| Dependency scan | `pip-audit`, `npm audit` |

---

## 5. Environment Variables

Secrets and runtime overrides — **never in YAML**.

| Variable | Required | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | yes | Claude API key |
| `OPENAI_API_KEY` | optional | OpenAI API key / embeddings |
| `SONARQUBE_TOKEN` | optional | Only if fetching issues live |
| `POSTGRES_URL` | yes | Relational DB DSN |
| `POSTGRES_PASSWORD` | yes (compose) | For the `postgres` service |
| `MONGO_URL` | yes | Telemetry DB DSN |
| `MONGO_PASSWORD` | yes (compose) | For the `mongo` service |
| `REDIS_URL` | yes | Queue + cache + pub/sub |
| `REDIS_PASSWORD` | yes (compose) | For the `redis` service |
| `SESSION_SECRET` | yes | Signed session token secret |
| `PAT_ENCRYPTION_KEY` | yes | KEK for user-supplied GitHub PATs at rest |
| `ZERO_DEBT_CONFIG` | yes | Path to `zero-debt.yaml` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | optional | Trace export target |
| `LOG_LEVEL` | optional | Overrides `logging.level` |
| `ZTIAP_*` | optional (v2) | ZTIAP integration |

> **Never put a `*_TOKEN`, `*_KEY`, `*_SECRET`, `*_PASSWORD` in `zero-debt.yaml`.** A startup check rejects YAMLs containing those key patterns.

---

## 6. Phase × Tool Matrix

| Phase | External services | Internal tools | Optional extras |
|---|---|---|---|
| **P1 — Sonar Fix** | GitHub, LLM, SonarQube (opt) | `GitHubTool`, `GitTool`, `FSTool`, `SonarTool`, `LanguageTool`, `PatchTool`, `ReportTool`, `TelemetrySink`, `LLMGateway` | tree-sitter, per-language native tooling |
| **P2 — Unit Tests** | GitHub, LLM | `GitHubTool`, `GitTool`, `FSTool`, `LanguageTool` (Java/Python/Rust strategies), `ReportTool`, `TelemetrySink`, `LLMGateway` | coverage tools per language |
| **P3 — Code Review** | GitHub, LLM | `GitHubTool`, `FSTool`, `LanguageTool`, `VectorStore`, `ReportTool`, `TelemetrySink`, `LLMGateway` | — |
| **P4 — PR Auto-Fix** | GitHub, LLM | `GitHubTool`, `GitTool`, `FSTool`, `LanguageTool`, `PatchTool`, `VectorStore`, `ReportTool`, `TelemetrySink`, `LLMGateway` | — |
| **P5 — E2E Playwright** | GitHub, LLM, target app | `GitHubTool`, `GitTool`, `FSTool`, `PlaywrightTool`, `ReportTool`, `TelemetrySink`, `LLMGateway` | `playwright-runner` service, browser binaries |
