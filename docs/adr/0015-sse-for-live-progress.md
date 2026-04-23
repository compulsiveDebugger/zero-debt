# ADR-0015: Server-Sent Events for Live Run Progress

**Status:** Accepted

## Context

The UI needs to show live progress for in-flight runs: which node is currently executing, how many Sonar issues are processed, LLM call counts, intermediate outcomes. Options:

1. **Polling** (`GET /runs/{id}` every few seconds). Simple, but wasteful (most responses are unchanged), and updates are laggy. Gets worse the more runs and users exist.
2. **WebSockets.** Full duplex. Overkill — the UI never sends messages *into* the run; it only observes. Requires extra infra (WS-capable proxy, sticky sessions, auth on upgrade).
3. **Server-Sent Events (SSE).** One-way stream over HTTP. Native browser support (`EventSource`). Works cleanly through most reverse proxies. Feels right for a status-update feed.

## Decision

Use **Server-Sent Events** for live run progress. Specifically:

- Endpoint: `GET /api/sse/runs/{run_id}` — `text/event-stream`, cookie-authenticated.
- Authorization: the run's `user_id` must match the session's `user_id` (or the session has an admin role). Rejected with 403 otherwise.
- Format: standard `event: <name>\ndata: <json>\n\n` frames. Named events: `node_start`, `node_end`, `progress`, `warning`, `error`, `terminal`.
- Heartbeat: a comment line (`:heartbeat\n\n`) every 15s so proxies don't idle-timeout the connection.
- Reconnect: clients supply `Last-Event-ID` on reconnect; server resumes from the next event in the Redis stream (`zerodebt:run:{id}:stream`) rather than starting over.

**Source of events:** LangGraph nodes publish events to Redis via a thin `publish_event(run_id, kind, payload)` helper. The API's SSE endpoint subscribes (via `XADD` / `XREAD` streams or classic `SUBSCRIBE`) and forwards. Workers never talk to the browser directly.

**Backpressure:** per-connection buffer is bounded. Slow clients that fall too far behind are disconnected with a `terminal` event; they can reopen and resume via `Last-Event-ID`.

## Consequences

**Positive**

- Lightweight — plain HTTP, no protocol upgrade, no WS libraries.
- Native browser reconnect semantics handle flaky networks.
- Proxies and enterprise firewalls generally pass SSE through without special config.
- One-way is the right shape for the use case.

**Negative**

- Each active stream holds an HTTP connection open. At the enterprise scale we target (low hundreds of concurrent runs), trivially absorbed by one API replica; at K8s scale we'll need to confirm the ingress can handle long-lived connections.
- SSE doesn't support binary payloads (not needed here).

**Discipline required**

- Authorization on every SSE connection open — this is not optional and not a one-time check. Session revocation mid-stream closes the connection.
- Events must be JSON-serializable; no raw LangGraph state dumps. Payloads go through a per-event schema.
