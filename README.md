# zero-debt

> Enterprise in-house **web app** that hunts down code debt — SonarQube findings, missing tests, weak PRs, visual regressions — on GitHub repos users connect, and opens branches that fix them.

**Status:** Architecture phase. Phase 1 (SonarQube Fixer) implementation kicking off.

---

## What it is

zero-debt is a **self-hosted, multi-user internal tool**. Corporate users log in, connect their GitHub repositories, configure per-repo settings, and trigger one of five AI-driven workflows. Every workflow runs asynchronously against the repo, streams progress back to the UI, and lands its results as a branch + PR on GitHub. **It never silently mutates `main`.**

- **Not** a GitHub App. It runs under your infrastructure.
- **Not** a CLI-first tool (though the orchestrator can still be invoked directly for tests).
- **Angular SPA** front end, **FastAPI** backend, async worker fleet, LangGraph orchestration.

## Capability phases

| Phase | Capability | Status |
|---|---|---|
| **P1** | SonarQube Recommendation Fixer | Blueprint complete — implementation next |
| **P2** | Unit Test Case Generator — **Java, Python, Rust** | Contract defined |
| **P3** | Comprehensive Code Reviewer (security / performance / style / architecture modes) | Contract defined |
| **P4** | PR Recommendation Auto-Fixer | Contract defined |
| **P5** | End-to-End Playwright Visual Regression Suite Generator | Contract defined |

All five phases share a single orchestration backbone (LangGraph), a single repository intelligence layer, a single LLM gateway, and a single delivery pipeline. Phases plug in as independently-compilable subgraphs — adding a phase does **not** require orchestrator changes.

---

## Architectural principles

1. **Subgraphs, not a monolith.** Every phase is an independently-testable LangGraph subgraph.
2. **Opaque phase payloads.** Orchestrator never learns phase-specific schemas.
3. **Logical model ids.** Nodes request `code-smart` or `review-deep`; the LLM Gateway resolves to Claude / OpenAI.
4. **Structural validation beats self-eval.** Patches gate on AST + lint + optional compile — never "ask the LLM if it's right."
5. **Never write to the default branch.** `allow_default_branch=false` by default.
6. **Secrets via env only.** No token ever in YAML, code, or logs.
7. **API never runs a phase inline.** Every phase runs in a worker behind an async queue.
8. **Auth is pluggable.** DB-based now, ZTIAP later, without code churn.

---

## Tech stack

| Layer | Choice |
|---|---|
| **UI** | Angular 17+ served via nginx |
| **API** | FastAPI (async), Pydantic v2 |
| **Orchestration** | LangGraph |
| **Workers** | Arq (async Redis queue) running LangGraph subgraphs |
| **Relational DB** | Postgres 16 (users, repos, runs, settings, LangGraph checkpointer) |
| **Telemetry / archive** | MongoDB (telemetry, LLM interaction archives, audit logs) |
| **Cache / Queue / Pub-Sub** | Redis |
| **Semantic index** (post-P1) | pgvector on Postgres, abstracted so Qdrant/Weaviate is a config swap |
| **Auth (v1)** | DB-based: Argon2 hashing, server-side sessions in Redis |
| **Auth (later)** | ZTIAP via the same `AuthProvider` interface |
| **LLM** | Anthropic Claude primary, OpenAI alternate — via `LLMGateway` abstraction |
| **Live UI updates** | Server-Sent Events (worker → API via Redis pub/sub → browser) |

---

## Deployment path

| Stage | Target |
|---|---|
| **Demo / budget approval** | Docker Compose on Rocky Linux 9 (single host) |
| **Post-approval** | Enterprise Rancher portal (Kubernetes) + ZTIAP auth + managed Postgres / Mongo / Redis |

The Compose file and the eventual Helm chart share the same container images; service boundaries match service boundaries. Migration is a packaging change, not a code change. See [docs/adr/0018-deployment-path.md](docs/adr/0018-deployment-path.md) once ZTIAP integration work begins.

---

## Documentation

| Doc | Purpose |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Functional + technical architecture, tiers, state schema, graph topology, LLM Gateway, persistence matrix, auth, async execution |
| [docs/PHASE-1-BLUEPRINT.md](docs/PHASE-1-BLUEPRINT.md) | Node-by-node SonarQube Fixer plan, error matrix, prompt strategy |
| [docs/TOOLS.md](docs/TOOLS.md) | Complete inventory: external services, runtime dependencies, internal tools, operator tooling |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Milestone sequencing (M0 → M10), effort estimates, risks |
| [docs/adr/](docs/adr/) | Architecture Decision Records |

---

## Quickstart (planned)

> Not runnable yet — scaffold lands in M0. This is the shape of the interface.

```bash
# 1. Configure secrets
export GITHUB_APP_TOKEN=...             # optional — for service-account repo access
export ANTHROPIC_API_KEY=sk-ant-...
export POSTGRES_PASSWORD=...
export MONGO_PASSWORD=...
export REDIS_PASSWORD=...
export SESSION_SECRET=...

# 2. Start the stack
docker compose up -d

# 3. Open the UI
open http://localhost:8080

# 4. In the UI:
#    - Sign up (DB auth for now)
#    - Connect a GitHub repo (paste a PAT)
#    - Upload or point at a SonarQube report
#    - Pick "Sonar Fixer" and click Run
#    - Watch live progress via SSE; grab the PR link from the summary
```

---

## Non-goals (for now)

- **No cross-org multi-tenancy.** One deployment = one corporate installation.
- **No in-tool fine-tuning or continuous learning.**
- **No GitHub App installation flow.** OAuth or PAT only.
- **No automatic commits to default branches. Ever.**

---

## Runtime targets

- **OS:** Rocky Linux 9 (v1), Kubernetes via Rancher (v2)
- **Python:** 3.11+
- **Node:** 20+ (for Angular build)
- **Browsers:** latest Chromium / Firefox (for P5 Playwright worker)

See [docs/TOOLS.md](docs/TOOLS.md) for the complete list.

---

## License

TBD — to be decided before the first public release.
