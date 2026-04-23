# ADR-0018: Deployment Path — Docker Compose → Rancher + ZTIAP

**Status:** Accepted

## Context

zero-debt has two distinct deployment stages driven by business reality:

1. **Stage 1 — Demo / budget approval.** Runs on a Rocky Linux 9 host via Docker Compose. Uses `DbAuthProvider`. In-cluster Postgres, Mongo, Redis. Goal: prove value, secure budget.
2. **Stage 2 — Enterprise production.** Moves to the corporate **Rancher** Kubernetes portal. **ZTIAP** provides auth. Postgres / Mongo / Redis become managed or enterprise-provisioned services.

We must not architect ourselves into a rewrite for Stage 2.

## Decision

**Container image parity + service boundary parity across stages.** The same container images that run under Compose run under Rancher. The same service split (`api`, `worker`, `web-ui`, `postgres`, `redis`, `mongo`, `playwright-runner`) becomes Deployments / StatefulSets.

Specifically:

- **Compose today.** `docker-compose.yml` is the source of truth for Stage 1 topology. Services communicate by name on a single bridge network. Secrets via env vars from a gitignored `.env` file.
- **Helm chart later (milestone DE-1).** Generated manually but structurally mirroring Compose. Each Compose service maps 1:1 to a K8s resource. Stateful services become StatefulSets; `web-ui`, `api`, `worker` become Deployments. Secrets injected via Kubernetes Secrets (later, external secret operators integrated with the corporate vault).
- **Auth flip (milestone DE-2).** `config.auth.provider` flips from `db` to `ztiap`. `ZtiapAuthProvider` uses ZTIAP session introspection; Redis still fronts it as a cache. Existing DB-auth users are either migrated or retired; that decision is separate. `DbAuthProvider` stays compiled into the image for local dev and tests even after the production flip.
- **Managed datastores (milestone DE-3).** `postgres`, `mongo`, `redis` services removed from the chart when managed equivalents are provisioned. DSN env vars point at the managed endpoints. No code change.

## Consequences

**Positive**

- Stage 1 → Stage 2 is a packaging and config shift, not a rewrite.
- Local development parity — every engineer runs the full stack under Compose regardless of what prod looks like.
- ZTIAP integration is localized to a single new class behind an existing interface.
- Datastore migration is a DSN change + data migration; no application code churn.

**Negative**

- We must resist "just this once" K8s-specific code paths creeping in pre-DE-1. If a feature requires K8s primitives, it waits.
- Stage-1 secrets management via env vars is coarse; mitigated by limiting Stage-1 to approved demo environments.

**Guardrails**

- Any Compose change that can't be replicated in K8s is rejected in review.
- Any K8s change introduced during DE-1 that can't be simulated under Compose requires an ADR addendum explaining the divergence.
- `DbAuthProvider` is kept as a first-class, tested implementation — even post-ZTIAP — so CI and local dev don't require a ZTIAP instance.
