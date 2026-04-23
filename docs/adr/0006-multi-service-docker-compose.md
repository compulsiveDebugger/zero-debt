# ADR-0006: Multi-Service Docker Compose over Single Container

**Status:** Accepted

## Context

zero-debt runs locally via Docker Compose (per the Rocky Linux 9 operational constraint). The deployment topology needs to accommodate:

- A long-running Postgres instance for the LangGraph checkpointer.
- An optional heavy Playwright runner (~1.5 GB image) for Phase 5 only.
- An optional log-aggregation sidecar for production-ish setups.

Options considered:

1. **Single container** — everything (app + Postgres + browsers) in one image. Simpler compose file, much larger image, coupled lifecycle, unsafe for any restart scenario.
2. **Multi-service compose** — one service per concern. Standard pattern, slightly more config, vastly better operational properties.

## Decision

Use multi-service Docker Compose:

| Service | Role | Notes |
|---|---|---|
| `zero-debt-core` | CLI / FastAPI + LangGraph runtime | Rocky 9 base, slim |
| `checkpoint-db` | Postgres 16 | Persistent volume; survives core restarts |
| `log-sink` (optional) | Vector / Fluent Bit | Absent → stdout to docker logs |
| `playwright-runner` | Headless browsers | `profiles: ["phase5"]` — only started when P5 runs |

## Consequences

**Positive**

- Checkpointer state outlives container restarts — resumability actually works.
- P1–P4 runs don't pay the 1.5 GB cost of pulling the Playwright image.
- Each service has its own restart policy and health check.
- Clean separation for future scaling (e.g., running `zero-debt-core` on Kubernetes with an external Postgres).

**Negative**

- More moving parts; operators must understand Compose profiles.
- Inter-service networking must be configured correctly (all services on `zero-debt-net`).
- Test harness must spin up Postgres in CI for integration tests — handled by a `docker-compose.test.yml` override.

**Explicit non-decisions**

- We do **not** yet split `zero-debt-core` into a worker + API pair. Single process handles both; if load justifies a split later, the LangGraph runtime and its stateless nature make that refactor local.
- We do **not** run the LLM provider as a service. We call external APIs; self-hosted LLMs are out of scope for v1.
