# zero-debt — Implementation Roadmap

> From empty repo to Phase 1 GA in the full web-app shape (UI + API + workers + multi-user + telemetry), then phase-by-phase expansion. Effort numbers are engineering-weeks at one full-time contributor.

---

## Phase 1 Delivery Path

| Milestone | Scope | Effort | Gate to next |
|---|---|---|---|
| **M0 — Monorepo skeleton** | Backend scaffold (`pyproject.toml`, `Dockerfile.api`, `Dockerfile.worker`), UI scaffold (`web/`, Angular 17, `Dockerfile`, `nginx.conf`), `docker-compose.yml` with all seven services, Pydantic settings, CI bootstrap (`ruff`, `mypy`, `ng lint`, `pytest`, `ng test`) | 1.5 weeks | `docker compose up` boots all services; API `/healthz` green; UI loads blank page |
| **M1 — Auth + user/repo models** | `AuthProvider` interface + `DbAuthProvider` (Argon2 + Redis sessions), Alembic migrations for users / repo_connections / repo_settings / runs / run_artifacts / run_events, signup/login/logout API + Angular screens, PAT-encrypted-at-rest with `PAT_ENCRYPTION_KEY` | 2 weeks | User can sign up, log in, connect a GitHub repo, persist settings |
| **M2 — Orchestrator spine** | `ZeroDebtState`, StateGraph assembly, pre-nodes (`validate_inputs`, `clone_or_fetch_repo`, `build_repo_outline` stub), post-nodes, `phase_router`, structlog wiring, LangGraph Postgres checkpointer | 1.5 weeks | Dry-run invocation with a stub subgraph reaches END inside a worker |
| **M3 — Job queue + SSE** | Arq worker entry, `POST /runs` enqueue, `GET /sse/runs/{id}` subscription, Redis pub/sub bridge, progress events from every node, per-user / per-repo concurrency caps, hard-timeout wiring | 1.5 weeks | UI submits a stub run and shows live progress via SSE end-to-end |
| **M4 — RIL v1** | File index, language detection (incl. Rust), manifest parser (Cargo, Maven/Gradle, Poetry, npm), minimal module graph | 1 week | Outline emitted for a Java + Python + Rust fixture repo |
| **M5 — LLM Gateway** | Abstract + Anthropic provider + model registry + JSON-schema enforcement + prompt-cache handling + archive every call to MongoDB | 1 week | Golden-path LLM call passes tests via `respx` replay; archive lands in Mongo |
| **M6 — Phase 1 subgraph** | All 10 P1 nodes, validator (tree-sitter + native toolchains for Java/TS/Python), summary report writer | 2.5 weeks | Fixture run: 10 Sonar issues → ≥7 fixed on a real OSS repo |
| **M7 — Delivery + UI for P1** | Branch/commit/push, PR open, templated PR body; Angular screens for P1 (report upload, run submit, progress view, summary/PR link) | 1.5 weeks | User can submit P1 from UI and land a real PR on a test repo |
| **M8 — Telemetry + audit** | `TelemetrySink` writing to Mongo collections (`run_events`, `llm_interactions`, `audit`), retention/rotation, operator "run detail" view in UI with event timeline | 1 week | Full audit trail for a completed run visible and searchable |
| **M9 — Harden P1** | Error matrix coverage, loop guards, golden-file tests, chaos tests for API failures, token-budget enforcement, session security review, PAT encryption review | 1.5 weeks | SLOs met: ≥80% fix rate on curated Sonar benchmark; 0 broken commits; security review signed off |
| **M10 — Phase contract freeze** | Stubs for P2–P5 subgraphs wired into router; UI phase-picker showing "coming soon" for disabled phases; doc of interface contract | 0.5 week | Flipping a phase flag routes to stub without orchestrator or UI shell edits |

**Total to P1 GA: ~15 weeks** from empty repo, inclusive of UI, auth, queue, telemetry.

---

## Phase 2–5 Expansion

Each subsequent phase runs 2–3 weeks at similar cadence. They land in whatever order business priorities dictate because the orchestrator contract is frozen at M10.

| Phase | Effort | Notes |
|---|---|---|
| **P2 — Unit Test Generator (Java / Python / Rust)** | 3–4 weeks | Three `LanguageTool` strategies: Java (JUnit 5 + Mockito), Python (PyTest), Rust (`cargo test` / `#[test]`). Coverage delta via JaCoCo / coverage.py / `cargo-tarpaulin`. Worker image grows to bundle JDK + cargo. |
| **P3 — Code Reviewer** | 2–3 weeks | Review-mode prompt packs (security / performance / style / architecture). GitHub PR review submission. First consumer of `VectorStore` for similar-code retrieval — pgvector index lands here. |
| **P4 — PR Auto-Fixer** | 2 weeks | Reuses P1 propose→validate→apply machinery. New: review-comment parser, conversation-thread state. |
| **P5 — E2E Playwright Generator** | 3 weeks | `playwright-runner` service activated. Visual baseline capture. User-flow inference from route configs. Biggest net-new surface. |

---

## Deployment Evolution

| Milestone | Scope | Effort | Gate |
|---|---|---|---|
| **DE-1 — Rancher packaging** | Helm chart mirroring Compose; values.yaml; secret injection via Kubernetes Secrets; horizontal worker scaling | 2 weeks | Stack runs on a Rancher cluster with identical behavior to Compose |
| **DE-2 — ZTIAP integration** | `ZtiapAuthProvider` implementation; session bridge; logout integration; migration path for existing DB users | 1.5 weeks | Auth provider config flip moves users from DB to ZTIAP with no code change |
| **DE-3 — Managed datastores** | Terraform modules for managed Postgres / Mongo / Redis; connection string injection; backup policy | 1 week | Stateful services no longer in-cluster |

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **LLM proposes broken code that passes diff-apply but fails compile** | High | High | Mandatory structural validation gate (AST + lint + optional compile). Never commit un-validated patches. |
| **Prompt drift across providers makes JSON output unreliable** | Medium | High | `json_schema` enforcement where supported; strict parser with one structured-retry on failure; gateway capability probe. |
| **Patch conflicts when multiple issues touch the same file** | High | Medium | Process one file atomically per LLM call; schema forces per-issue patch separation. |
| **Sonar reports with stale line numbers** | Medium | Medium | Tolerance window ±3 lines; skip if symbol doesn't match rule pattern. |
| **Runaway token spend** | Medium | High | Per-run `token_budget`; gateway enforces; `status=PARTIAL` instead of hard-fail. |
| **LangGraph checkpointer state grows unbounded** | Low | Medium | TTL sweep on completed-run checkpoints after 30 days. |
| **Phase 2–5 assumptions not expressible in current state schema** | Medium | High | Opaque `phase_input`/`phase_output` dicts + per-phase Pydantic models at the boundary. |
| **Accidental pushes to `main`** | Low | Catastrophic | `allow_default_branch=false` default; branch-name convention; unit test asserts refusal. |
| **Secrets leaked in logs or Mongo archives** | Low | High | Structlog redactor; `include_llm_prompts=false` default for prompts containing sensitive repo content; CI `gitleaks` scan. |
| **Worker image bloat from per-language toolchains** | High (after P2) | Medium | Track image size in CI; if it exceeds 2 GB, split into per-language sidecar runners. |
| **Long-running runs stuck in queue during restart** | Medium | Medium | LangGraph checkpointer + Arq at-least-once semantics; UI shows "resumed" state. |
| **PAT secrets leak via compromised Postgres backup** | Low | High | Encrypt PATs with `PAT_ENCRYPTION_KEY` at application layer; backups are then useless without the KEK. |
| **MongoDB grows unbounded from LLM archives** | Medium | Medium | Collection-level TTL (`llm_archive_retention_days` from config); operator dashboard shows collection sizes. |
| **SSE connections piling up on slow clients** | Low | Low | Bounded per-run buffer; disconnect policy; monitoring counter on active SSE count. |

---

## Open Decision Points

Carried forward to the relevant milestone kickoff:

1. **Worker image shape (M6 → P2 kickoff).** Fat image with all toolchains vs. per-language sidecars invoked over a local socket. Fat image is the default; revisit if image exceeds 2 GB or cold-start becomes painful.
2. **LLM batching inside `propose_fix` (M6).** All issues for one file in one LLM call vs. issue-by-issue. Default: **batch-per-file**, with schema forcing per-issue patch separation.
3. **Validator backend mix (M6).** tree-sitter breadth vs. language-native precision. Default: **tree-sitter for MVP**, language-native for Java/TS/Rust post-M6.
4. **Vector store backend (P3).** pgvector default; swap to Qdrant or Weaviate if index size or query latency pushes past pgvector's comfort zone (roughly millions of vectors).
5. **UI streaming: SSE vs. WebSockets (M3).** SSE default — one-way, simpler, proxies better. Revisit only if a real use case for duplex communication appears.
6. **PR review auto-close behavior (P3).** Default: **comment only**. Humans decide the formal review state.
7. **GitHub auth: PAT vs. OAuth App (M1 → DE-phase).** PAT for v1 (simplest). OAuth app with device flow considered once DE-1 lands.

---

## Non-Goals for v1

Explicit to prevent scope drift:

- No cross-org multi-tenancy from a single deployment.
- No cross-phase composition in a single run (one phase per run; compose at the caller level).
- No fine-tuning or continuous-learning loops.
- No automatic fixes for Sonar rules flagged as "requires human judgment."
- No mobile UI.
- No GitHub App installation flow.
