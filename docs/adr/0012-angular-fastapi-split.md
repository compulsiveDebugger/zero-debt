# ADR-0012: Angular SPA + FastAPI Backend Split

**Status:** Accepted

## Context

zero-debt needs a UI ([ADR-0011](0011-web-hosted-multi-user-service.md)). Two broad shapes:

1. **Monolithic server-rendered app** (e.g., FastAPI + Jinja + htmx, or Django). Simpler deployment, but weaker fit for live progress streams and rich per-phase forms; and the enterprise is Angular-aware.
2. **SPA + JSON API.** Clear separation; SPA handles UX, API handles data. Scales cleanly when new frontends (admin dashboards, mobile web) are added later.

Framework choice for the SPA:

- **Angular** — enterprise default; strong forms / HTTP / RxJS story; dependency injection familiar to Java/.NET teams.
- React — more common in startups; less opinionated.
- Vue / Svelte — smaller footprint but less enterprise traction.

Backend choice:

- **FastAPI** — matches our async-first Python stack, first-class Pydantic integration, native OpenAPI generation, async SSE support via `sse-starlette`.
- Django / DRF — heavier, sync-first, worse fit for SSE + async workers.

## Decision

- Frontend: **Angular 17+** SPA, built with Angular CLI, served by **nginx** in the `web-ui` container.
- Backend: **FastAPI** in the `api` container. All endpoints JSON except SSE streams.
- `nginx` proxies `/api/*` and `/sse/*` to the API service; everything else is static SPA assets.
- OpenAPI schema is generated from FastAPI and consumed by the Angular client (types + an HTTP client) via `openapi-generator` or equivalent — no hand-rolled duplicate types.

## Consequences

**Positive**

- Clean separation of concerns; UI team and API team can iterate independently.
- FastAPI + Pydantic means the API's contract types are the same Pydantic models we already use for LLM and external data — zero duplication.
- SSE support is first-class in FastAPI; we don't need a reverse proxy trick.
- Auto-generated TypeScript client keeps the frontend in sync with API changes, enforced in CI.

**Negative**

- Two build pipelines (Python + Node). More CI stages; more image layers.
- Angular has a learning curve for teams without TypeScript/RxJS experience.
- The API and UI versions can drift if the generated client isn't regenerated on every backend change — mitigated by a CI check.

**Discipline required**

- The OpenAPI spec is the contract. Don't work around it with hand-written types on the UI side "for speed." Regenerate the client.
