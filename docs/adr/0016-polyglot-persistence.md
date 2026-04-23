# ADR-0016: Polyglot Persistence — Postgres + Redis + MongoDB + pgvector

**Status:** Accepted

## Context

zero-debt has persistence needs that don't fit cleanly into a single store:

| Data class | Characteristics |
|---|---|
| Users, repo connections, settings, runs | Relational, referentially-constrained, transactional. Low volume. |
| LangGraph checkpointer | Transactional JSON blobs keyed by run. Low-to-medium volume. |
| Job queue, sessions, pub/sub, concurrency counters | Ephemeral, high churn, TTL-driven. |
| Run events (full history), audit, LLM prompt/response archive | Append-heavy, schema-flexible, large. Queried by `run_id` / `user_id` / date. |
| Semantic code embeddings | Vector data; nearest-neighbor queries. |

Forcing all of this into Postgres would work but creates real problems:

- LLM archives are large and churn-heavy — they'd bloat the operational database and complicate backups.
- Sessions + queue in Postgres is possible (`LISTEN/NOTIFY`, `SKIP LOCKED`) but Redis does this with far less tuning.
- Vector search in Postgres is *possible* via `pgvector` — and that's the one case where co-locating makes sense.

## Decision

Adopt polyglot persistence, with clearly scoped roles:

| Store | Owns | Why |
|---|---|---|
| **Postgres 16 (+ pgvector)** | Users, repos, settings, runs, artifacts, LangGraph checkpointer, **semantic index** (via pgvector) | Source of truth; transactional; already needed. Vector extension rides along for free. |
| **Redis 7** | Job queue (Arq), sessions, pub/sub, concurrency counters, response cache | Right tool for ephemeral + pub/sub. |
| **MongoDB 7** | Telemetry events (full history), LLM interaction archives, audit trail | Append-heavy, schema-flexible, high volume — wrong fit for the operational Postgres. |

Data-flow rules:

1. **Postgres is source of truth.** If a datum is referenced by another datum, it lives in Postgres.
2. **Mongo is append-only operational archive.** Nothing references Mongo documents by ID from Postgres; Mongo holds snapshots keyed by `run_id` / `user_id` / `date`.
3. **Redis holds nothing that can't be recomputed from Postgres + Mongo** (except in-flight sessions, which have TTL semantics already).
4. **Vector index is co-located on Postgres via pgvector** — see [ADR-0017](0017-vector-store-abstraction.md) for how it's abstracted.

Mongo collections:

- `run_events` — per-node start/end/progress events (full history, not the trimmed copy in Postgres).
- `llm_interactions` — request + response + token usage + latency per LLM call. Retention driven by `telemetry.llm_archive_retention_days`.
- `audit` — security-relevant events: login, logout, permission denials, PAT rotation, settings changes.

## Consequences

**Positive**

- Each data class lives where it's ergonomic to query and operate.
- Postgres stays small and fast — backups, restores, and migrations stay tractable.
- Mongo's schema flexibility absorbs the reality that LLM-interaction shape evolves quickly (new providers, new structured outputs).
- Redis handles exactly what it's best at.
- One vector store, no new service — pgvector on the existing Postgres.

**Negative**

- Four stores to operate. More ops surface than a monostore.
- No native cross-store joins — queries that span "runs" (Postgres) and "events" (Mongo) must be assembled in application code.
- Consistency is per-store. A run is marked `SUCCEEDED` in Postgres but its Mongo archive write might lag by seconds. Acceptable; tests and UI handle this with a brief "archiving..." state when needed.

**Discipline required**

- Never dual-write the same fact to Postgres and Mongo as "source of truth." Pick one. In practice: operational state → Postgres; operational *history* → Mongo.
- Resist the temptation to start querying Mongo from LangGraph nodes for speed. Nodes talk to the abstractions (`TelemetrySink`, `VectorStore`, `GitHubTool`, etc.), not raw clients.
