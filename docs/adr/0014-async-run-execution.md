# ADR-0014: Async Run Execution via Arq + Redis

**Status:** Accepted

## Context

A phase run takes minutes to tens of minutes (clone + outline + LLM loop + validate + push). The web request that triggers it absolutely cannot block on it:

- HTTP timeouts (load balancers, reverse proxies, browsers) are far shorter than a typical run.
- API instances need to stay free for concurrent user traffic.
- Workers restart / redeploy independently from the API tier.
- Users close browsers and come back; runs must survive that.

We need a job queue. Options considered:

1. **Celery.** Mature, ubiquitous. Sync-first; async support added later and feels bolted-on. Heavy broker / result backend setup. Overkill for our scale.
2. **RQ.** Simple, Redis-backed. Sync-only at its core.
3. **Arq.** Async-native Python job queue on Redis. Clean fit for our FastAPI + asyncio stack. Lightweight.
4. **Temporal.** Durable workflow engine. Impressive but massive operational footprint and conceptually competes with LangGraph for "workflow state."
5. **Postgres-backed queue** (`pgmq`, SKIP LOCKED). No new service, but lacks pub/sub for the progress streaming we also need ([ADR-0015](0015-sse-for-live-progress.md)).

## Decision

Use **Arq** as the job queue, **Redis** as the broker. One FastAPI-side enqueue; N `worker` containers consuming.

Contract:

- `POST /runs` validates input, persists a `RUN` row (`status=pending`) in Postgres, then enqueues `{run_id}` on Arq (Redis queue `zerodebt:runs`).
- A worker picks up `run_id`, loads run + user + repo from Postgres, invokes the LangGraph orchestrator with a fully-populated `ZeroDebtState`.
- Worker publishes progress events to Redis pub/sub channel `zerodebt:run:{run_id}` from each node; API's SSE endpoint subscribes.
- Terminal status is persisted in Postgres and published as a final SSE event.
- LangGraph checkpointer (Postgres, [ADR-0007](0007-postgres-checkpointer.md)) means a killed worker resumes from the last checkpoint.

**Concurrency & safety:**

- Per-user and per-repo concurrency caps enforced at enqueue time via Redis counters.
- Hard run timeout (`run.run_hard_timeout_minutes`, default 45). LangGraph checks a deadline between nodes — cooperative cancellation.
- Arq task is idempotent at the `run_id` level — repeated dequeue does not re-create side effects; checkpointer handles restart replay.

## Consequences

**Positive**

- API responses return in milliseconds regardless of phase complexity.
- Workers scale horizontally (`docker compose up --scale worker=N` / K8s replicas).
- Redis is already needed for sessions and pub/sub — zero incremental infra cost.
- Async-native throughout — no sync/async impedance mismatch.
- Crash recovery is free via checkpointer + at-least-once delivery.

**Negative**

- Arq is less feature-rich than Celery (no scheduled tasks, limited chord/group). Acceptable — we don't need them.
- Redis becomes a critical single point of failure. Mitigated by planning for a managed / HA Redis at DE-3 (see ROADMAP).
- Debugging an in-flight run means finding the worker pod handling it — requires good observability.

**Discipline required**

- Workers must stay stateless between tasks. No in-process caches that outlive a single run.
- Progress-event publishing must not be on the critical path — a Redis hiccup should log a warning, not fail a run. Handled in the publish call's error boundary.
