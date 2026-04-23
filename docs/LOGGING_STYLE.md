# Logging Style Guide

Structured JSON logs are an interface. Treat them like an API contract.

---

## 1. Why structured

- **Machines parse them** — log aggregator queries, alert rules, dashboards.
- **Humans skim them** — `jq` filters in dev; SRE filters in incident response.
- **Context survives async boundaries** — contextvars carry `run_id`, `user_id`, `phase` across awaits.

---

## 2. Tooling

- Python: **`structlog`** with JSON renderer. Configured once in `observability/logging.py`.
- Angular: a thin wrapper around `console.*` that emits JSON in production builds and pretty in dev. Cross-context `request_id` propagated via interceptor.

Both stacks emit the **same key vocabulary** so log aggregation can join across tiers.

---

## 3. Log line shape

Every line is a JSON object with these mandatory fields:

| Key | Type | Example | Notes |
|---|---|---|---|
| `timestamp` | ISO-8601 UTC | `"2026-04-23T14:02:18.422Z"` | Millisecond precision, `Z` suffix |
| `level` | string | `"info"` | `debug` / `info` / `warning` / `error` / `critical` |
| `event` | string | `"patch_applied"` | snake_case, **noun_verb_past_tense** describing what happened |
| `service` | string | `"api"` / `"worker"` / `"web"` | The emitting tier |
| `version` | string | `"0.4.2"` | Build version |

Plus context (when available) and the per-event payload.

---

## 4. Event names

**Format:** `<subject>_<verb-past-tense>` — describe an event that **already happened**, not a thing being done.

| ✅ Good | ❌ Bad |
|---|---|
| `repo_connected` | `connect_repo` |
| `patch_applied` | `applying_patch` |
| `llm_request_sent` | `calling_llm` |
| `validation_gate_failed` | `gate_failed` (which gate?) |
| `session_expired` | `session-expired` (no kebab in events) |

Event names are part of the contract. Renaming an event breaks dashboards and alerts. **Treat renames like API breaks.**

---

## 5. Standard context keys

These are reserved across the system. Use them consistently — never invent a different name for the same concept.

| Key | When | Source |
|---|---|---|
| `request_id` | Every API request, every LLM call | Generated at request entry; propagated via `X-Request-ID` |
| `run_id` | Anything inside a run | Bound at task start in worker |
| `user_id` | Any user-scoped action | Bound from session principal |
| `repo_connection_id` | Anything tied to a connected repo | Bound at run start |
| `phase` | Phase-scoped events | Bound at subgraph entry |
| `node` | LangGraph node events | Bound by the node middleware |
| `iteration` | Loop iteration counter | For loop nodes |
| `error_code` | `error`/`critical` lines | Match the API error envelope's `code` |
| `duration_ms` | Anything timed | Integer milliseconds |
| `tool` | Tool-layer ops (`github_tool`, `git_tool`, ...) | Bound by tool middleware |
| `op` | Tool sub-operation (`clone`, `push`, `pr_create`) | |
| `latency_ms` | External API timings | Integer |
| `tokens_in`, `tokens_out`, `tokens_total` | LLM calls | Integer |
| `model` | LLM calls | Resolved concrete model id |
| `model_role` | LLM calls | Logical role (`code-smart`, etc.) |
| `cache_status` | LLM calls | `"hit"` \| `"miss"` \| `"none"` |

---

## 6. How to log — Python

```python
from structlog import get_logger

log = get_logger(__name__)

# At task entry, bind context that propagates via contextvars:
log = log.bind(run_id=run_id, user_id=user_id, phase="sonar_fix")

log.info("repo_clone_started", repo_url=repo_url)
log.info("repo_clone_completed", duration_ms=8_140, files=4_218)

log.warning(
    "validation_gate_failed",
    gate="ast_parse",
    file="payment/Tokenizer.java",
    detail="unexpected token } at line 142",
    iteration=2,
    iteration_cap=3,
)

try:
    ...
except LLMSchemaError as e:
    log.error(
        "llm_schema_error",
        error_code="LLM_INVALID_JSON",
        request_id=e.request_id,
        attempt=e.attempt,
        exc_info=True,        # includes stack trace as a field, not formatted
    )
    raise
```

**Rules:**

- Pass values as kwargs. **Never f-string into the event name or message.**
- Don't pre-format strings. Numbers stay numbers, durations stay integer ms, IDs stay strings.
- Use `exc_info=True` for caught exceptions you log; structlog records the trace structurally.
- Never mutate a log entry after emit.

---

## 7. How to log — Angular

```ts
import { Logger } from '@app/core/logging';

const log = new Logger('runs.progress');

log.info('sse_connected', { runId, lastEventId });
log.warn('sse_disconnected', { runId, reason: err.message, willReconnect: true });
log.error('run_submit_failed', { runId, errorCode: e.code, status: e.status });
```

The `Logger` wrapper:
- Adds `service: "web"`, `version`, `request_id` (from interceptor), `user_id` (from `AuthService`).
- In dev, pretty-prints. In prod, single-line JSON.
- Forwards `error` and `critical` to a backend endpoint `POST /api/_logs` (rate-limited).

---

## 8. Levels — when to use each

| Level | Use for |
|---|---|
| `debug` | Verbose diagnostic. **Off in prod by default.** Never required for understanding system state. |
| `info` | Normal events worth recording: state transitions, completed operations, external calls. |
| `warning` | Recoverable anomalies: retries, fallbacks, rate-limits, validation rejections that lead to skip-not-fail. |
| `error` | A unit of work failed (a request, a node, a job). System still healthy. |
| `critical` | The process or system is impaired. Pages someone. |

Specifically:

- **`info`**, not `debug`, for the major node start/end events. They're load-bearing for ops.
- **`warning`**, not `info`, for any retry — including LLM retries and patch validation retries.
- **`error`**, not `warning`, when a run terminates `failed`.
- **`critical`** for: DB connection lost, KEK missing on startup, queue unreachable.

---

## 9. Secret redaction — last line of defense

Configured in `observability/logging.py`. The redactor:

- Drops keys matching `(?i).*(token|key|secret|password|authorization).*`.
- Replaces values matching common secret patterns (GitHub PAT, AWS keys, JWT) with `***`.
- **Always treat the redactor as a backstop, not the primary defense.** Don't pass tokens to log calls in the first place.

There is a unit test that asserts a synthetic PAT in a log call gets redacted. **It must stay green forever.**

---

## 10. PII

We log very little PII by design.

- **Email:** logged only on auth events, and only as `user_id` (the UUID), never the email itself in operational logs. Auth events are an exception (`user_email_verified`) but live in the audit collection only.
- **PR titles / commit messages:** may be logged. They're already public on GitHub.
- **Patch contents:** never logged. The full diff lives in the PR; the log records `file`, `lines_added`, `lines_removed`.

---

## 11. LLM prompts

- Disabled by default (`logging.include_llm_prompts: false`).
- When enabled, prompts and responses go to the **Mongo `llm_interactions` archive only**, not to stdout logs.
- Prompts may carry sensitive repo content. Make this an opt-in, retention-bounded write.

---

## 12. Sampling and volume

- High-volume events (e.g., per-issue `issue_ranked`) are **sampled** if their per-run count exceeds 100 — log every Nth, plus the first and last.
- Per-run summary events are always logged in full.
- `error` and `critical` are never sampled.

---

## 13. Anti-patterns

| ❌ | ✅ |
|---|---|
| `log.info(f"applied patch for {file}")` | `log.info("patch_applied", file=file)` |
| `log.error("Bad token")` | `log.error("auth_token_invalid", error_code="AUTH_INVALID_TOKEN")` |
| `print("debugging X")` | `log.debug("x_observed", x=...)` |
| `log.info("OK")` | Pick a real event name |
| `log.info(f"user logged in: {email}")` | `log.info("user_logged_in", user_id=principal.user_id)` |
| `log.error(token=token)` | Never log the token; log the request_id |
| Catch + swallow + log nothing | Catch + log at WARN+ + re-raise (or handle deliberately) |

---

## 14. What good logs look like

A successful run produces a log timeline like:

```
{"timestamp":"...","event":"run_started","run_id":"r_7a2","phase":"sonar_fix",...}
{"timestamp":"...","event":"repo_clone_started","repo_url":"..."}
{"timestamp":"...","event":"repo_clone_completed","duration_ms":8140,"files":4218}
{"timestamp":"...","event":"repo_outline_built","duration_ms":3820,"languages":{"java":0.67,"kotlin":0.22,...}}
{"timestamp":"...","event":"sonar_report_loaded","issues":47}
{"timestamp":"...","event":"node_started","node":"propose_fix","iteration":1}
{"timestamp":"...","event":"llm_request_sent","model_role":"code-smart","tokens_in":4812,"request_id":"r_7a2_001"}
{"timestamp":"...","event":"llm_response_received","tokens_out":1184,"latency_ms":4200,"cache_status":"hit"}
{"timestamp":"...","event":"validation_gates_passed","file":"order/RefundService.java"}
{"timestamp":"...","event":"patch_applied","file":"order/RefundService.java","issues_closed":2}
...
{"timestamp":"...","event":"run_completed","status":"succeeded","duration_ms":1124000,"issues_fixed":41,"issues_skipped":6}
```

If you can't picture how `jq 'select(.event=="patch_applied")'` would answer "which files did this run touch?" — your event names are wrong.
