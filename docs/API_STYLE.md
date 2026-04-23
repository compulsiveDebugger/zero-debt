# API Style Guide

REST conventions for the FastAPI backend. The OpenAPI spec generated from FastAPI is the contract; this document is what informs the choices that go into the spec.

---

## 1. Versioning

- All endpoints under `/api/`. We do not version in the URL today (single deployment, single client).
- Breaking changes to a v1 API will introduce `/api/v2/...` and deprecate v1 with a 6-month sunset.

---

## 2. Resource naming

- **Plural nouns:** `/api/repos`, `/api/runs`, `/api/users`. Never `/api/run` for a list.
- **Kebab-case** for multi-word resources: `/api/repo-connections` if we ever have one.
- **No verbs in URLs.** Use HTTP methods. Exception: domain operations that aren't CRUD (`/api/runs/{id}/cancel`).
- **Nested routes for clear ownership:** `/api/repos/{id}/settings/{phase}`. Don't nest more than 2 levels deep.

---

## 3. HTTP methods

| Method | Use |
|---|---|
| `GET` | Fetch. **Idempotent. No side effects.** |
| `POST` | Create OR a non-idempotent action |
| `PUT` | Replace a resource entirely. **Idempotent.** |
| `PATCH` | Partial update |
| `DELETE` | Remove. **Idempotent.** |

Never use `GET` with side effects (no triggering jobs from GET — that breaks proxies and crawlers).

---

## 4. Status codes

| Code | Use |
|---|---|
| `200 OK` | Successful read or update |
| `201 Created` | Resource created (sync) |
| `202 Accepted` | Accepted for async processing (use for `POST /api/runs`) |
| `204 No Content` | Success, no body (logout, delete) |
| `400 Bad Request` | Generic validation failure (rare; prefer 422) |
| `401 Unauthorized` | Missing or invalid auth |
| `403 Forbidden` | Authenticated but not allowed |
| `404 Not Found` | Resource does not exist OR is scoped away from this principal |
| `409 Conflict` | State conflict (e.g., terminal run cannot cancel) |
| `422 Unprocessable Entity` | Request body validation failure (FastAPI default) |
| `429 Too Many Requests` | Rate limit / concurrency cap |
| `500 Internal Server Error` | Server bug |
| `503 Service Unavailable` | Dependency down (e.g., Mongo unreachable) |

**Use 404 instead of 403** for resources scoped to other users — avoid information leakage about resource existence.

---

## 5. Request and response shapes

### 5.1 Successful response

Either a resource object or a list envelope. **No top-level wrapping for single resources.**

```jsonc
// GET /api/repos/abc-123
{
  "id": "abc-123",
  "github_url": "https://github.com/...",
  "default_branch": "main",
  "created_at": "2026-04-23T12:01:58Z"
}
```

### 5.2 List envelope

Always paginated. Cursor-based. Keys: `items`, `next_cursor`, `has_more`.

```jsonc
// GET /api/runs?limit=20&cursor=...
{
  "items": [
    { "id": "r_7a2f8c1e", "phase": "sonar_fix", "status": "succeeded", ... },
    ...
  ],
  "next_cursor": "eyJjcmVhdGVkX2F0Ijoi...",
  "has_more": true
}
```

### 5.3 Error response

Always wrapped under `error`. Stable shape:

```jsonc
{
  "error": {
    "code": "RUN_CONCURRENCY_LIMIT",
    "message": "User is at the per-user concurrency cap of 2 runs.",
    "detail": {
      "current_active_runs": 2,
      "limit": 2
    },
    "request_id": "req_01HJ..."
  }
}
```

- `code` — `UPPER_SNAKE_CASE`. Stable across versions; never reword.
- `message` — human-readable. Safe to show in UI toasts.
- `detail` — optional structured payload for the UI to use; never PII.
- `request_id` — propagated from `X-Request-ID` request header (or generated). Echoed in `X-Request-ID` response header too.

FastAPI's default 422 validation errors are wrapped to this envelope by a global exception handler.

---

## 6. Field naming

- **`snake_case` for JSON keys** in API. Yes, even though JS prefers camelCase — the OpenAPI generator handles the conversion in the Angular client. Backend stays Pythonic.
- **Timestamps:** `*_at`, ISO-8601 UTC with `Z` suffix (`"2026-04-23T12:01:58Z"`). Always tz-aware. Never epoch milliseconds.
- **IDs:** UUID v4 strings; field name `id` for self, `<resource>_id` for foreign references.
- **Booleans:** `is_*` / `has_*` / `should_*` prefixes; never `flag`/`enabled` without context (`run_enabled` is fine; `enabled` alone is not).
- **Counts:** `*_count` (`runs_count`, not `n_runs`).
- **Enums:** lowercase string values matching the Python enum value (`"sonar_fix"`, `"succeeded"`).

---

## 7. Pagination

- **Cursor-based, not page-based.** Page numbers shift under inserts.
- `limit` query param, default 20, max 100.
- `cursor` query param, opaque string. Server encodes whatever it needs (we use `(created_at, id)` keyset).
- Response includes `next_cursor` and `has_more`; absent `next_cursor` ⇒ end of list.

---

## 8. Filtering and sorting

- **Filtering** via query params: `?phase=sonar_fix&status=succeeded&since=2026-04-01T00:00:00Z`.
- **Sorting** via `?sort=field` with `-` prefix for descending: `?sort=-created_at`.
- **Multiple values:** repeat the param: `?status=succeeded&status=partial`.
- **Search:** `?q=...` for free-text (when supported); not all resources support search.

---

## 9. Authentication & authorization

- **Cookie-based session auth** for the SPA. `zd_session` cookie, `HttpOnly; Secure; SameSite=Lax`.
- **CSRF token** required on state-changing endpoints. Header `X-CSRF-Token`; matched against the `zd_csrf` cookie.
- **Bearer tokens** are not used in v1 (no machine-to-machine API).
- **Authorization checks at the endpoint**, not in middleware. Even if middleware checks, the endpoint re-asserts.

---

## 10. SSE endpoints

- **Path convention:** `/api/sse/<resource>/{id}` (e.g., `/api/sse/runs/{id}`).
- **Content type:** `text/event-stream`.
- **Authentication:** same cookie auth as REST endpoints.
- **Authorization:** principal must own the resource (or have admin role).
- **Frame format:** standard SSE — `event: <name>\ndata: <json>\n\n`.
- **Heartbeat:** `:heartbeat\n\n` every 15s.
- **Reconnect support:** server honors `Last-Event-ID` request header.

See [ADR-0015](adr/0015-sse-for-live-progress.md).

---

## 11. Idempotency

- `PUT` and `DELETE` are inherently idempotent.
- `POST` operations that create exclusively (e.g., `POST /api/runs`) accept an optional `Idempotency-Key` header. Server caches the response under that key for 24h. Repeat calls return the cached response.
- `POST /api/runs/{id}/cancel` is idempotent: repeat calls on a cancelled run return 200, repeat on a terminal-non-cancelled run returns 409.

---

## 12. Rate limiting

- Per-IP on auth endpoints: 5 failed attempts per 15 minutes → 429.
- Per-user on run enqueue: governed by concurrency caps (see [ADR-0014](adr/0014-async-run-execution.md)).
- 429 responses include `Retry-After` header (seconds) and the standard error envelope.

---

## 13. Concurrency / locking

- **Optimistic concurrency** for `PATCH` of mutable resources via `If-Match` ETag header. Resource responses include `ETag`.
- **No long-held locks.** Workers serialize via the run-id-as-thread-id pattern in LangGraph.

---

## 14. Caching

- **GET responses:** `Cache-Control: no-store` by default. APIs are not browser-cacheable; the SPA owns its caching.
- **Static resources** (UI bundle): aggressive immutable caching with content-hash filenames.

---

## 15. Webhooks (future)

For inbound webhooks (P3 trigger from GitHub), when implemented:

- Path: `/api/webhooks/<source>` (e.g., `/api/webhooks/github`).
- HMAC signature verification on every request.
- Idempotency key from the source's delivery id.
- Async processing via Arq; immediate `200 OK` to the source.

---

## 16. OpenAPI generation

- FastAPI's `/openapi.json` is the source of truth.
- The Angular client is regenerated from this spec. CI fails if the committed client is out of sync.
- All endpoints have `response_model=`, `responses={...}`, and a docstring summary.
- All Pydantic models have field descriptions where the field name isn't self-explanatory.

---

## 17. Anti-patterns

- ❌ `POST /api/runs/create` — verb in URL.
- ❌ `GET /api/runs?action=cancel&id=...` — GET with side effect.
- ❌ Returning `200 OK` with `{"success": false}` — use the right status code.
- ❌ Different error envelopes per endpoint.
- ❌ Sending JWTs in cookies (we use opaque session tokens; revocation is real).
- ❌ Server-side rendering — backend serves JSON only; UI handles all rendering.
