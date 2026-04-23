# Architecture Decision Records

ADRs capture the "why" behind load-bearing architectural choices. Each ADR is short, self-contained, and dated. Supersede instead of editing.

## Format

Each ADR uses this structure:

- **Status** — Proposed / Accepted / Superseded by ADR-XXXX
- **Context** — what forced the decision
- **Decision** — what we chose
- **Consequences** — positive and negative outcomes we accept

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-langgraph-stategraph-as-backbone.md) | LangGraph `StateGraph` as orchestration backbone | Accepted |
| [0002](0002-subgraph-per-phase-isolation.md) | One independently-compilable subgraph per phase | Accepted |
| [0003](0003-opaque-phase-payloads.md) | `phase_input` / `phase_output` are opaque dicts in shared state | Accepted |
| [0004](0004-llm-gateway-abstraction.md) | Single `LLMGateway` with logical model ids | Accepted |
| [0005](0005-structural-patch-validation.md) | Patch validation is structural, never LLM self-eval | Accepted |
| [0006](0006-multi-service-docker-compose.md) | Multi-service Docker Compose over single container | Accepted |
| [0007](0007-postgres-checkpointer.md) | Postgres-backed LangGraph checkpointer in prod | Accepted |
| [0008](0008-repository-intelligence-snapshot.md) | Repository Intelligence is a frozen per-run snapshot | Accepted |
| [0009](0009-yaml-plus-env-config.md) | YAML for structure + env vars for secrets | Accepted |
| [0010](0010-no-default-branch-writes.md) | Never write to default branches by default | Accepted |
| [0011](0011-web-hosted-multi-user-service.md) | Web-hosted multi-user service (not a GitHub App, not CLI-first) | Accepted |
| [0012](0012-angular-fastapi-split.md) | Angular SPA + FastAPI backend split | Accepted |
| [0013](0013-pluggable-authentication.md) | Pluggable authentication: DB today, ZTIAP later | Accepted |
| [0014](0014-async-run-execution.md) | Async run execution via Arq + Redis | Accepted |
| [0015](0015-sse-for-live-progress.md) | Server-Sent Events for live progress | Accepted |
| [0016](0016-polyglot-persistence.md) | Polyglot persistence: Postgres + Redis + MongoDB + pgvector | Accepted |
| [0017](0017-vector-store-abstraction.md) | Vector store abstraction (pgvector default, Qdrant later) | Accepted |
| [0018](0018-deployment-path-docker-to-rancher-ztiap.md) | Deployment path: Docker Compose → Rancher + ZTIAP | Accepted |

## When to add a new ADR

- A choice has plausible alternatives and you can name the tradeoffs.
- The decision changes a contract other code depends on.
- You find yourself re-explaining a choice in PR reviews.

## When *not* to add an ADR

- It's a style preference covered by linters.
- It's a tactical implementation detail with no consumer outside the file.
- It's already documented in code comments or CLAUDE.md.
