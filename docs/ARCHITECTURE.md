# zero-debt — Architecture

> Canonical technical reference. Phase 1 details live in [PHASE-1-BLUEPRINT.md](PHASE-1-BLUEPRINT.md). Decision rationale lives in [adr/](adr/).

---

## 0. Deployment Model & User-Facing Shape

zero-debt is a **self-hosted enterprise web application**, not a GitHub App and not a CLI. A corporate user opens the UI, authenticates, connects one or more GitHub repositories, configures per-repo settings, and submits runs of capability phases. Runs execute asynchronously on a worker fleet; the UI streams progress over SSE and renders the final report.

Deployment evolves:

| Stage | Target |
|---|---|
| **Demo / budget approval** | Docker Compose on Rocky Linux 9 |
| **Enterprise production** | Rancher / Kubernetes + ZTIAP auth + managed datastores |

Container images and service boundaries are identical across stages. Migration is a packaging/config shift, not a code shift. See [ADR-0011](adr/0011-web-hosted-multi-user-service.md) and [ADR-0018](adr/0018-deployment-path-docker-to-rancher-ztiap.md).

---

## 1. Functional Architecture

### 1.1 System Overview

Four conceptual tiers:

1. **UI tier** — Angular SPA. Presents auth, repo connection, settings, run submission, live progress, reports.
2. **API tier** — FastAPI. Handles auth, CRUD for users/repos/settings/runs, run enqueue, SSE stream endpoints. **Never runs a phase inline.**
3. **Execution tier** — Arq workers consuming a Redis queue. Each worker runs the LangGraph orchestrator end-to-end for one run.
4. **Integration tier** — GitHub / Anthropic / SonarQube / target application (P5). All accessed via internal tool abstractions.

Horizontally across all tiers:
- **Ingestion & Intelligence Layer (RIL)** — clones the repo once per run, builds a language/framework fingerprint.
- **Delivery Layer** — commits, pushes branches, opens PRs, emits reports.
- **Observability plumbing** — structured logs to stdout + MongoDB telemetry sink, OTel traces, per-node timings.

### 1.2 High-Level Topology

```mermaid
flowchart LR
    U[User Browser<br/>Angular SPA] -->|HTTPS| NG[nginx]
    NG -->|static| U
    NG -->|/api/*| API[FastAPI]
    NG -->|/sse/*| API

    API -->|CRUD| PG[(Postgres<br/>users, repos, runs,<br/>LG checkpointer)]
    API -->|enqueue| RQ[(Redis<br/>queue + cache + sessions<br/>+ pub/sub)]
    API -->|subscribe| RQ
    API -->|telemetry write| MG[(MongoDB<br/>telemetry / LLM archive / audit)]
    API -->|auth| AUTH[AuthProvider<br/>Db → ZTIAP]

    RQ -->|dequeue| W1[Arq Worker 1]
    RQ -->|dequeue| W2[Arq Worker N]

    W1 --> LG[LangGraph<br/>Orchestrator]
    W2 --> LG
    LG --> PG
    LG --> MG
    LG -->|publish| RQ
    LG -->|HTTPS| GH[GitHub API]
    LG -->|HTTPS| LLM[LLM Gateway<br/>→ Anthropic / OpenAI]
    LG -->|HTTPS| SQ[SonarQube<br/>optional]

    subgraph Phase5only[Phase 5 only]
      PW[playwright-runner]
    end
    W1 -. spawn for P5 .-> PW
```

### 1.3 Multi-User Data Model (relational, Postgres)

```mermaid
erDiagram
    USER ||--o{ REPO_CONNECTION : owns
    USER ||--o{ RUN : triggers
    REPO_CONNECTION ||--o{ REPO_SETTING : has
    REPO_CONNECTION ||--o{ RUN : scopes
    RUN ||--o{ RUN_ARTIFACT : produces
    RUN ||--o{ RUN_EVENT : emits

    USER {
        uuid id PK
        string email UK
        string password_hash
        string display_name
        string role
        timestamptz created_at
        timestamptz last_login_at
    }
    REPO_CONNECTION {
        uuid id PK
        uuid user_id FK
        string github_url
        string default_branch
        string pat_secret_ref
        timestamptz created_at
    }
    REPO_SETTING {
        uuid id PK
        uuid repo_connection_id FK
        string phase
        jsonb config_override
    }
    RUN {
        uuid id PK
        uuid user_id FK
        uuid repo_connection_id FK
        string phase
        string status
        jsonb input_payload
        jsonb output_payload
        string work_branch
        int pr_number
        string pr_url
        timestamptz created_at
        timestamptz finished_at
    }
    RUN_ARTIFACT {
        uuid id PK
        uuid run_id FK
        string kind
        string uri
    }
    RUN_EVENT {
        uuid id PK
        uuid run_id FK
        string kind
        jsonb payload
        timestamptz at
    }
```

Operational + source-of-truth data lives in Postgres. **High-volume telemetry, LLM prompt/response archives, and audit trails live in MongoDB** — see §2.11.

### 1.4 Repository Intelligence Layer (RIL)

The single most important shared primitive. Every phase reads; no phase writes.

| Artifact | Contents |
|---|---|
| `repo_outline.files` | Flat file index — language, LOC, path, last-modified |
| `repo_outline.languages` | Language mix with percentages |
| `repo_outline.build_systems` | Maven, Gradle, npm, Poetry, Cargo, Go modules, etc. |
| `repo_outline.frameworks` | Spring, React, Django, FastAPI, etc. |
| `repo_outline.test_frameworks` | JUnit, PyTest, `cargo test`, Jest, Playwright |
| `repo_outline.entry_points` | `main` classes, HTTP routers, CLI entry defs |
| `repo_outline.module_graph` | Lightweight dependency graph between top-level packages |
| `repo_outline.conventions` | Indent style, naming, lint config presence |

Frozen per run — see [ADR-0008](adr/0008-repository-intelligence-snapshot.md). Semantic indexing is a separate concern (§2.12).

### 1.5 Phase Module Contracts

| Phase | Input | Output | Trigger | Dependencies |
|---|---|---|---|---|
| **P1 — SonarQube Fixer** | `SonarReport` (JSON or live API) + `RepoRef` | Branch + `FixSummaryReport` + PR | UI / API | RIL, `GitHubTool`, `LLMGateway`, `SonarTool`, `LanguageTool` |
| **P2 — Unit Test Generator (Java / Python / Rust)** | `RepoRef` + optional `TargetFileGlob` | Branch + `CoverageDeltaReport` + PR | UI / API | RIL, `GitHubTool`, `LLMGateway`, `LanguageTool` (per-language strategy) |
| **P3 — Code Reviewer** | `PRRef` + `ReviewMode` (security / performance / style / architecture / all) | PR review comments + `ReviewSummary` | UI / webhook | RIL, `GitHubTool`, `LLMGateway`, review-mode prompt pack |
| **P4 — PR Auto-Fixer** | `PRRef` + review comments (P3 or human) | Additional commits + `AutoFixAuditLog` | UI, after P3 or manually | RIL, `GitHubTool`, `LLMGateway`, `PatchTool` |
| **P5 — E2E Playwright Generator** | `RepoRef` + `AppBaseURL` + optional `UserFlowsDoc` | Branch + Playwright suite + `VisualBaselineSet` + PR | UI / API | RIL, `GitHubTool`, `LLMGateway`, `PlaywrightTool`, `playwright-runner` |

Universal preconditions: auth'd user, connected repo, RIL built, writable workspace. Universal postconditions: run is terminal (success / failure / partial), structured log emitted, telemetry persisted.

### 1.6 User Interaction Flow

```mermaid
sequenceDiagram
    actor User
    participant UI as Angular SPA
    participant API as FastAPI
    participant PG as Postgres
    participant Q as Redis Queue
    participant W as Arq Worker
    participant LG as LangGraph Orchestrator
    participant GH as GitHub

    User->>UI: Log in
    UI->>API: POST /auth/login
    API->>PG: verify user (via AuthProvider)
    API-->>UI: session cookie

    User->>UI: Connect repo, configure settings
    UI->>API: POST /repos, PATCH /repos/:id/settings
    API->>PG: persist

    User->>UI: Run P1 (Sonar Fix)
    UI->>API: POST /runs {phase:sonar_fix, repo_id, inputs}
    API->>PG: INSERT run (status=pending)
    API->>Q: enqueue(run_id)
    API-->>UI: {run_id, status_url, sse_url}

    UI->>API: GET /sse/runs/:id  (EventSource)
    API-->>UI: subscribes to Redis pub/sub zerodebt:run:{id}

    Q-->>W: dequeue run_id
    W->>LG: invoke(state)
    loop per node
        LG->>Q: publish progress event
        Q-->>API: pub/sub → SSE to UI
    end
    LG->>GH: create branch, commit, open PR
    LG->>PG: write run_output, artifacts
    LG-->>W: done
    W->>Q: publish terminal event
    Q-->>API: final SSE
    API-->>UI: close stream

    User->>UI: Open report
    UI->>API: GET /runs/:id
    API->>PG: fetch
    API-->>UI: summary + PR link
```

---

## 2. Technical Architecture

### 2.1 LangGraph Graph Topology (execution-tier)

Unchanged by the new tiers — the LangGraph orchestrator still owns one run end-to-end. It runs inside an Arq worker instead of a CLI process.

```mermaid
flowchart TD
    START([START]) --> V[validate_inputs]
    V --> CL[clone_or_fetch_repo]
    CL --> RIL[build_repo_outline]
    RIL --> R{phase_router}

    R -->|sonar_fix| S1[SonarFixSubgraph]
    R -->|unit_test_gen| S2[UnitTestSubgraph]
    R -->|code_review| S3[CodeReviewSubgraph]
    R -->|pr_fix| S4[PRFixSubgraph]
    R -->|e2e_test_gen| S5[E2ESubgraph]

    S1 --> POST[post_phase_normalize]
    S2 --> POST
    S3 --> POST
    S4 --> POST
    S5 --> POST

    POST --> D{delivery_required?}
    D -->|yes| PUSH[commit_and_push_branch]
    D -->|no| REP
    PUSH --> PR[open_pull_request]
    PR --> REP[generate_summary_report]
    REP --> LOG[emit_structured_log]
    LOG --> END([END])

    V -. on_error .-> FAIL[fail_fast_handler]
    CL -. on_error .-> FAIL
    FAIL --> LOG
```

Every node also publishes a progress event to Redis (`zerodebt:run:{run_id}`) so the API can forward it as SSE. See [ADR-0015](adr/0015-sse-for-live-progress.md).

### 2.2 AgentState Schema

```python
from __future__ import annotations
from typing import TypedDict, Optional, List, Dict, Any
from enum import Enum


class AgentPhase(str, Enum):
    REPO_ANALYSIS   = "repo_analysis"
    SONAR_FIX       = "sonar_fix"
    UNIT_TEST_GEN   = "unit_test_gen"
    CODE_REVIEW     = "code_review"
    PR_FIX          = "pr_fix"
    E2E_TEST_GEN    = "e2e_test_gen"


class RunStatus(str, Enum):
    PENDING    = "pending"
    RUNNING    = "running"
    SUCCEEDED  = "succeeded"
    FAILED     = "failed"
    PARTIAL    = "partial"


class RepoRef(TypedDict):
    url:        str
    branch:     str
    commit_sha: Optional[str]


class ZeroDebtState(TypedDict, total=False):
    # Identity & config (immutable across the run)
    run_id:          str          # matches runs.id in Postgres
    user_id:         str          # who triggered it (authz + audit)
    repo_connection_id: str
    active_phase:    AgentPhase
    llm_model_id:    str
    config:          Dict[str, Any]

    # Repo context (RIL writes; read-only after)
    repo_ref:        RepoRef
    repo_local_path: Optional[str]
    repo_outline:    Optional[Dict[str, Any]]

    # Phase I/O (opaque — ADR-0003)
    phase_input:     Dict[str, Any]
    phase_output:    Dict[str, Any]

    # Delivery artifacts
    work_branch:     Optional[str]
    pr_number:       Optional[int]
    pr_url:          Optional[str]

    # Observability & control
    status:          RunStatus
    errors:          List[Dict[str, Any]]
    warnings:        List[str]
    iteration_count: int
    phase_iteration: int
    llm_token_usage: Dict[str, int]
    timings_ms:      Dict[str, int]
```

`user_id` and `repo_connection_id` are new — they tie every run back to the relational model for authorization, audit, and the UI's "my runs" list.

### 2.3 Node Inventory

| Node Name | Responsibility | Async? | Tools Used |
|---|---|---|---|
| `validate_inputs` | Schema-validate, resolve per-repo config override | No | Pydantic |
| `clone_or_fetch_repo` | Clone or `git fetch` | Yes | `GitTool`, `GitHubTool` |
| `build_repo_outline` | Run RIL indexers | Yes | `FSTool`, `LanguageDetector`, `ManifestParser` |
| `phase_router` | Branch on `active_phase` | No | — |
| `post_phase_normalize` | Coerce `phase_output` for delivery | No | — |
| `commit_and_push_branch` | Create branch, commit, push | Yes | `GitTool`, `GitHubTool` |
| `open_pull_request` | Open PR with templated body | Yes | `GitHubTool` |
| `generate_summary_report` | Human + JSON report | No | `ReportTool` |
| `emit_structured_log` | JSON-log terminal state to stdout + MongoDB | No | `StructLogger`, `TelemetrySink` |
| `fail_fast_handler` | Capture error, mark `status=FAILED` | No | — |
| **P1 subgraph** | see [PHASE-1-BLUEPRINT.md](PHASE-1-BLUEPRINT.md) | | |

### 2.4 Tool Layer Design

Stateless classes, async-first, registered in a `ToolRegistry` sourced from `state.config`. No node instantiates an SDK client directly. Catalog: [TOOLS.md](TOOLS.md).

```
tools/
  github_tool.py
  git_tool.py
  fs_tool.py
  sonar_tool.py
  language_tool.py       # per-language strategies: Java, Python, Rust, TS, Go...
  patch_tool.py
  playwright_tool.py     # P5
  report_tool.py
  telemetry_sink.py      # Mongo writer for archives / audit
  vector_store.py        # abstracted; pgvector default, Qdrant/Weaviate later
```

Safety contracts (every tool):
- Filesystem writes jailed under `config.workspace_root`.
- GitHub mutations refuse default-branch targets unless `config.allow_default_branch=true` — [ADR-0010](adr/0010-no-default-branch-writes.md).
- No `shell=True`; argument arrays only.
- Structlog redactor strips known secret keys before emission.

### 2.5 LLM Gateway Abstraction

Unchanged; see [ADR-0004](adr/0004-llm-gateway-abstraction.md) for rationale.

```python
from abc import ABC, abstractmethod
from typing import AsyncIterator, Optional, Sequence, Literal
from pydantic import BaseModel


class LLMMessage(BaseModel):
    role:    Literal["system", "user", "assistant", "tool"]
    content: str
    name:    Optional[str] = None


class LLMRequest(BaseModel):
    model_id:     str
    messages:     Sequence[LLMMessage]
    temperature:  float = 0.0
    max_tokens:   int = 4096
    json_schema:  Optional[dict] = None
    tool_schemas: Optional[list[dict]] = None
    stop:         Optional[list[str]] = None
    request_id:   str


class LLMResponse(BaseModel):
    text:           str
    structured:     Optional[dict]
    tool_calls:     list[dict]
    finish_reason:  str
    usage:          dict
    latency_ms:     int
    raw_provider:   str


class LLMGateway(ABC):
    @abstractmethod
    async def complete(self, req: LLMRequest) -> LLMResponse: ...
    @abstractmethod
    async def stream(self, req: LLMRequest) -> AsyncIterator[str]: ...
    @abstractmethod
    def supports(self, capability: str) -> bool: ...
```

Every call is archived to MongoDB's `llm_interactions` collection (request + response + token usage + latency) for debugging, cost attribution, and replay.

### 2.6 Authentication Layer

Pluggable — see [ADR-0013](adr/0013-pluggable-authentication.md).

```python
from abc import ABC, abstractmethod
from pydantic import BaseModel


class UserPrincipal(BaseModel):
    user_id: str
    email:   str
    roles:   list[str]
    claims:  dict                # provider-specific extras


class AuthProvider(ABC):
    @abstractmethod
    async def login(self, credentials: dict) -> UserPrincipal: ...
    @abstractmethod
    async def verify_session(self, session_token: str) -> UserPrincipal: ...
    @abstractmethod
    async def logout(self, session_token: str) -> None: ...
```

**v1 (`DbAuthProvider`):** email + password with Argon2id hashing (`argon2-cffi`). Session tokens are opaque, stored in Redis (`session:{token}` → `UserPrincipal` JSON, TTL = configurable). Cookies are `HttpOnly; Secure; SameSite=Lax`. CSRF token on state-changing endpoints.

**v2 (`ZtiapAuthProvider`):** swaps the implementation; API layer calls the same interface. Redis session storage still applies as a cache in front of ZTIAP token introspection.

Passwords are **never** in YAML, logs, or Mongo archives. Structlog redactor plus request-level middleware scrubs `password`, `token`, `secret`, `authorization`.

### 2.7 Long-Running Execution Model

See [ADR-0014](adr/0014-async-run-execution.md).

**Flow.** API validates the run request, persists a `RUN` row with `status=pending`, enqueues `{run_id}` on Redis via Arq. One of N workers dequeues and invokes the LangGraph orchestrator. Every node publishes progress to the run-scoped Redis channel. API-side SSE endpoints subscribe and forward events. Terminal status is persisted in Postgres and also published as a final SSE event.

**Crash recovery.** Arq restart policy + LangGraph Postgres checkpointer ([ADR-0007](adr/0007-postgres-checkpointer.md)) means a killed worker resumes from the last checkpoint without redoing LLM work.

**Concurrency.** Per-user and per-repo concurrency limits enforced at enqueue time (Redis rate-limit counters). Prevents one user from monopolizing the worker fleet.

**Timeouts.** Hard run timeout (configurable, default 45 min). Arq cancellation is cooperative — LangGraph checks `state.config.deadline_at` between nodes.

### 2.8 Server-Sent Events

See [ADR-0015](adr/0015-sse-for-live-progress.md).

- Endpoint: `GET /api/sse/runs/{run_id}` (auth'd via session cookie).
- Authorization: the run's `user_id` must match the session's `user_id` (or user has admin role).
- Format: standard `text/event-stream` with `event:` + `data:` fields. Heartbeats every 15s.
- Source: the API subscribes to Redis pub/sub channel `zerodebt:run:{run_id}`. Workers publish; API forwards.
- Backpressure: SSE is one-way; slow clients are disconnected after a bounded buffer fills.

### 2.9 Docker Compose Strategy

Multi-service. See [ADR-0006](adr/0006-multi-service-docker-compose.md) and [ADR-0016](adr/0016-polyglot-persistence.md).

| Service | Purpose |
|---|---|
| `web-ui` | nginx serving the Angular build + proxying `/api/*` and `/sse/*` to `api` |
| `api` | FastAPI app (uvicorn) — CRUD, auth, SSE |
| `worker` | Arq worker(s) — runs LangGraph end-to-end per run |
| `postgres` | Postgres 16 + pgvector extension; users, repos, runs, LG checkpointer, vector index |
| `redis` | Queue + cache + pub/sub + sessions |
| `mongo` | Telemetry, LLM archives, audit logs |
| `playwright-runner` | Heavy browser image; `profiles: ["phase5"]` — only started when P5 runs |
| `log-sink` (optional) | Vector / Fluent Bit if external aggregation is wired up |

```yaml
services:
  web-ui:
    build: { context: ./web, dockerfile: Dockerfile }
    image: zero-debt/web:dev
    ports: ["8080:80"]
    depends_on: [api]
    networks: [zero-debt-net]

  api:
    build: { context: ., dockerfile: Dockerfile.api }
    image: zero-debt/api:dev
    environment: &core_env
      - GITHUB_TOKEN=${GITHUB_TOKEN:-}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY:-}
      - SONARQUBE_TOKEN=${SONARQUBE_TOKEN:-}
      - ZERO_DEBT_CONFIG=/app/config/zero-debt.yaml
      - POSTGRES_URL=postgresql://zerodebt:${POSTGRES_PASSWORD}@postgres:5432/zerodebt
      - MONGO_URL=mongodb://zerodebt:${MONGO_PASSWORD}@mongo:27017/zerodebt
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - SESSION_SECRET=${SESSION_SECRET}
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
      mongo:    { condition: service_healthy }
    networks: [zero-debt-net]

  worker:
    build: { context: ., dockerfile: Dockerfile.worker }
    image: zero-debt/worker:dev
    environment: *core_env
    volumes:
      - workspaces:/var/lib/zero-debt/workspaces
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    deploy:
      replicas: 2
    networks: [zero-debt-net]

  postgres:
    image: pgvector/pgvector:pg16           # Postgres 16 with pgvector preloaded
    environment:
      - POSTGRES_USER=zerodebt
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=zerodebt
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zerodebt"]
      interval: 5s
      retries: 10
    networks: [zero-debt-net]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
    volumes: [redisdata:/data]
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a ${REDIS_PASSWORD} ping | grep PONG"]
      interval: 5s
      retries: 10
    networks: [zero-debt-net]

  mongo:
    image: mongo:7
    environment:
      - MONGO_INITDB_ROOT_USERNAME=zerodebt
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
    volumes: [mongodata:/data/db]
    healthcheck:
      test: ["CMD","mongosh","--quiet","--eval","db.adminCommand('ping')"]
      interval: 10s
      retries: 10
    networks: [zero-debt-net]

  playwright-runner:
    profiles: ["phase5"]
    image: mcr.microsoft.com/playwright:v1.44.0-jammy
    volumes: [workspaces:/workspaces]
    networks: [zero-debt-net]

volumes:
  pgdata:
  redisdata:
  mongodata:
  workspaces:

networks:
  zero-debt-net: { driver: bridge }
```

### 2.10 Configuration Schema

YAML for structure + env for secrets. Pydantic v2 `BaseSettings`. See [ADR-0009](adr/0009-yaml-plus-env-config.md).

```yaml
# config/zero-debt.yaml
deployment:
  stage: docker-compose              # docker-compose | rancher
  public_base_url: http://localhost:8080

auth:
  provider: db                       # db | ztiap
  session_ttl_minutes: 480
  password_policy: { min_length: 12, require_mixed_case: true }

run:
  workspace_root: /var/lib/zero-debt/workspaces
  per_user_concurrent_runs: 2
  per_repo_concurrent_runs: 1
  run_hard_timeout_minutes: 45

llm:
  default_model_role: code-smart
  roles:
    fast:        { provider: anthropic, model: claude-haiku-4-5 }
    code-smart:  { provider: anthropic, model: claude-sonnet-4-6 }
    review-deep: { provider: anthropic, model: claude-opus-4-7 }
  prompt_cache: true
  temperature_default: 0.0

queue:
  backend: arq-redis
  default_queue: zerodebt:runs
  worker_concurrency: 4

telemetry:
  mongo_db: zerodebt_telemetry
  collections:
    events:         run_events
    llm:            llm_interactions
    audit:          audit
  llm_archive_enabled: true
  llm_archive_retention_days: 90

vector_store:
  backend: pgvector                  # pgvector | qdrant | weaviate (future)
  embedding_model: text-embedding-3-small
  dimensions: 1536

github:
  api_base: https://api.github.com
  allow_default_branch: false
  pr_template_path: templates/pr_body.md.j2

phases:
  sonar_fix:
    enabled: true
    max_issues_per_run: 50
    min_severity: MAJOR
    per_file_iteration_cap: 3
    validate_with_compile: true
  unit_test_gen:
    enabled: false
    languages: [python, java, rust]
    coverage_target_delta: 0.10
  code_review:  { enabled: false }
  pr_fix:       { enabled: false }
  e2e_test_gen: { enabled: false }

logging:
  level: INFO
  format: json
  include_llm_prompts: false
```

Secrets (env only): `GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `SONARQUBE_TOKEN`, `POSTGRES_PASSWORD`, `MONGO_PASSWORD`, `REDIS_PASSWORD`, `SESSION_SECRET`.

### 2.11 Persistence Matrix

See [ADR-0016](adr/0016-polyglot-persistence.md).

| Store | Owns | Access pattern | Schema |
|---|---|---|---|
| **Postgres** | Users, repo connections, settings, runs, run artifacts, run events (last N), LangGraph checkpointer | Read-heavy CRUD + transactional writes | Relational — migrated via Alembic |
| **Redis** | Job queue, sessions, cache, pub/sub channels, concurrency counters | Ephemeral / throughput-bound | Key-value + streams |
| **MongoDB** | Telemetry events (full history), LLM interactions archive, audit trail | Append-heavy; queried by run_id / user_id / date | Schema-flexible — per collection |
| **pgvector** (on Postgres) | Semantic code embeddings keyed by `(repo_connection_id, file_path, chunk_id)` | Nearest-neighbor queries from phase nodes | `vector(1536)` column indexed with `ivfflat` |

**Why not one store:** Postgres is the operational source of truth; mixing high-volume telemetry (millions of rows per run) into it creates I/O and backup pain. Mongo's flexible schema + append-optimized workload handles the archive tier. Redis handles the classes of data Postgres is bad at (ephemeral, high-churn).

**Why pgvector, not Qdrant, initially:** one fewer service. Semantic search isn't on the P1 critical path. The `VectorStore` abstraction ([ADR-0017](adr/0017-vector-store-abstraction.md)) makes a later swap a config change.

### 2.12 Semantic Code Index (post-P1)

- Indexing runs asynchronously on repo-connect and on significant mutations.
- Chunks files by function/class boundaries (tree-sitter-driven) rather than fixed windows.
- Embedding computed via `LLMGateway` (provider-specific `embeddings` capability) or a local `sentence-transformers` model for offline deployments.
- Used by P3 (find similar patterns) and P4 (retrieve prior review rationales). Not needed for P1.

---

## 3. Project Scaffold

Monorepo. Backend + frontend in one repo; separate Dockerfiles.

```
zero-debt/
├── pyproject.toml
├── Dockerfile.api
├── Dockerfile.worker
├── docker-compose.yml
├── README.md
├── config/
│   ├── zero-debt.yaml
│   └── prompt_packs/
│       ├── sonar_fix/
│       ├── unit_test/
│       ├── code_review/
│       ├── pr_fix/
│       └── e2e_gen/
├── src/zero_debt/
│   ├── api/                        # FastAPI
│   │   ├── main.py
│   │   ├── routers/
│   │   │   ├── auth.py
│   │   │   ├── users.py
│   │   │   ├── repos.py
│   │   │   ├── runs.py
│   │   │   └── sse.py
│   │   ├── dependencies.py         # auth, db, redis, mongo injection
│   │   └── middleware.py
│   ├── worker/                     # Arq entry
│   │   ├── main.py
│   │   └── tasks.py                # enqueue-able task fns (thin wrappers)
│   ├── auth/
│   │   ├── provider.py             # AuthProvider base
│   │   ├── db_provider.py
│   │   └── ztiap_provider.py       # stub
│   ├── db/
│   │   ├── models.py               # SQLAlchemy models
│   │   ├── session.py
│   │   └── migrations/             # Alembic
│   ├── mongo/
│   │   ├── client.py
│   │   └── sinks.py                # TelemetrySink, LLMArchiveSink, AuditSink
│   ├── redis/
│   │   ├── queue.py                # Arq config
│   │   └── pubsub.py
│   ├── settings.py                 # Pydantic BaseSettings
│   ├── state.py                    # ZeroDebtState TypedDict + enums
│   ├── graph/
│   │   ├── orchestrator.py
│   │   ├── routing.py
│   │   ├── pre_nodes.py
│   │   ├── post_nodes.py
│   │   └── subgraphs/
│   │       ├── sonar_fix.py
│   │       ├── unit_test.py        # stub
│   │       ├── code_review.py      # stub
│   │       ├── pr_fix.py           # stub
│   │       └── e2e_gen.py          # stub
│   ├── intelligence/
│   │   ├── outline.py
│   │   ├── language_detect.py
│   │   ├── manifest_parser.py
│   │   └── module_graph.py
│   ├── llm/
│   │   ├── gateway.py
│   │   ├── model_registry.py
│   │   └── providers/
│   │       ├── anthropic.py
│   │       └── openai.py
│   ├── tools/                      # see TOOLS.md
│   ├── vector/
│   │   ├── store.py                # VectorStore interface
│   │   └── backends/
│   │       ├── pgvector.py
│   │       └── qdrant.py           # stub
│   ├── models/                     # Pydantic API / LLM contracts
│   ├── phases/                     # per-phase pure logic (not LangGraph nodes)
│   ├── observability/
│   │   ├── logging.py
│   │   ├── tracing.py
│   │   └── metrics.py
│   └── persistence/
│       └── checkpointer.py         # LangGraph Postgres checkpointer factory
├── web/                            # Angular SPA
│   ├── angular.json
│   ├── package.json
│   ├── Dockerfile                  # builds UI + nginx
│   ├── nginx.conf
│   └── src/
│       ├── app/
│       │   ├── core/               # auth guard, http interceptor, services
│       │   ├── features/
│       │   │   ├── auth/           # login, signup
│       │   │   ├── repos/          # connect, list, settings
│       │   │   ├── runs/           # submit, progress (SSE), report
│       │   │   └── settings/       # user prefs
│       │   └── shared/
│       └── environments/
└── tests/
    ├── unit/
    ├── integration/                # API + worker + real Postgres/Redis/Mongo via compose.test
    ├── e2e/                        # Playwright against the UI
    └── fixtures/
```

---

## 4. Cross-Cutting Concerns

### 4.1 Observability

- **Logs:** structlog JSON to stdout; a processor also ships events to MongoDB `run_events` collection with `run_id`, `user_id`, `phase`, `node`, `timings_ms`, tagged error context. LLM prompts archived separately in `llm_interactions` — see telemetry config.
- **Traces:** OpenTelemetry spans around every async node and every LLM call. Exporter pluggable (OTLP default).
- **Metrics:** Prometheus endpoint on the API and each worker: run counters, queue depth, LLM token cost per phase.

### 4.2 Persistence & Resumability

LangGraph Postgres checkpointer ([ADR-0007](adr/0007-postgres-checkpointer.md)) plus Arq's retry semantics. Every node crossing an external boundary (`propose_fix`, GitHub mutation) is a checkpoint boundary.

### 4.3 Security Posture

- Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`. Session rotation on privilege change.
- CSRF token on state-changing endpoints; verified by middleware.
- All secrets via env; Argon2id for password hashing; per-tenant (future) encryption keys for PATs stored in Postgres.
- PATs at rest encrypted with a deployment-level KEK (env var); decrypted only inside workers.
- FSTool jailed under `workspace_root`; no path traversal.
- No `shell=True`; arg arrays only.
- GitHub token scopes: `repo` minimum.
- No `main`/`master` writes by default ([ADR-0010](adr/0010-no-default-branch-writes.md)).

### 4.4 Extensibility Contract for Phases 2–5

Adding a phase requires:

1. Pydantic input/output models under `models/`.
2. A subgraph under `graph/subgraphs/<phase>.py` exporting `compile_subgraph() -> CompiledStateGraph`.
3. A routing case in `graph/routing.py` — the *only* orchestrator change.
4. A `phases.<phase>` config block with `enabled` flag.
5. A prompt pack under `config/prompt_packs/<phase>/`.
6. UI surface: a phase-specific form component under `web/src/app/features/runs/` that POSTs the right `phase_input` shape.

No changes to `ZeroDebtState`, pre/post nodes, LLM Gateway, or auth layer.
