# Testing Strategy

What we test, why we test, how we test, what we don't.

The aim is **confidence per minute of CI time**. We're not chasing 100% line coverage; we're chasing fast feedback that catches the bugs we'd otherwise ship.

---

## 1. Test pyramid

| Tier | Count target | Speed | What it covers |
|---|---|---|---|
| **Unit** | ~70% of tests | <1s each | Pure logic, transformations, validators, ranking, prompt assembly, individual services |
| **Integration** | ~25% of tests | 1–10s each | API endpoints with real Postgres/Redis/Mongo via Compose; LangGraph subgraph end-to-end with mocked LLM |
| **E2E** | ~5% of tests | 10s+ each | Critical user paths in Playwright against the running stack |

We do **not** chase a coverage number. We chase **specific, named risks** (see [03_scope_timeline_risks.md](03_scope_timeline_risks.md) §3).

---

## 2. Coverage gates (CI-enforced)

| Module | Minimum line coverage |
|---|---|
| `zero_debt/auth/` | 95% |
| `zero_debt/tools/github_tool.py` (branch-write guards) | 100% |
| `zero_debt/graph/` | 85% |
| `zero_debt/phases/` | 85% |
| `zero_debt/llm/` | 80% |
| `zero_debt/api/` | 80% |
| Everything else | 70% (project floor) |
| Frontend `core/` and `shared/` | 80% |
| Frontend feature components | 70% |

A PR that drops a gated module below its threshold fails CI. To raise a threshold, **raise it permanently in the same PR** that adds the tests.

---

## 3. Backend testing (Python)

### 3.1 Tooling

- **`pytest`** — the only runner. No `unittest.TestCase`.
- **`pytest-asyncio`** mode `strict` — every async test marked.
- **`respx`** — HTTP mocking (Anthropic, GitHub).
- **`testcontainers`** *or* **Compose service** — real Postgres/Redis/Mongo for integration.
- **`freezegun`** — time-sensitive tests.
- **`hypothesis`** — property-based tests for ranking, schema validation.

### 3.2 Layout

```
tests/
  unit/               ← pure logic, no I/O. 70% of tests.
    auth/
    graph/
    phases/sonar_fix/
    llm/
    tools/
  integration/        ← real datastores, mocked external APIs.
    api/              ← FastAPI TestClient + ASGITransport
    worker/           ← Arq task end-to-end
    graph/            ← Subgraph against fixture repo + mocked LLM
  e2e/                ← rare; full stack via docker-compose.test.yml
  fixtures/
    repos/            ← committed fixture repos (small, deterministic)
    sonar_reports/    ← committed Sonar JSON exports
    llm_recordings/   ← respx replay snapshots
```

### 3.3 Naming and structure

- **One test file per source file:** `validate_patch.py` ↔ `tests/unit/graph/test_validate_patch.py`.
- **Test names describe behavior:**
  - ✅ `test_validate_patch_rejects_diff_outside_reported_line_window`
  - ❌ `test_case_3`, `test_validate_patch_works`
- **Arrange / Act / Assert** structure with blank lines between sections.
- **One concept per test.** Multiple `assert` lines are fine when they together describe one behavior.

```python
async def test_propose_fix_retries_once_on_invalid_json():
    # Arrange
    gateway = FakeGateway(responses=[INVALID_JSON, VALID_FIX_PROPOSAL])
    state = make_state_with_one_issue()

    # Act
    new_state = await propose_fix(state, gateway=gateway)

    # Assert
    assert new_state["phase_input"]["proposal"] is not None
    assert gateway.call_count == 2
```

### 3.4 Fixtures

- **Module-scoped** for expensive setup (DB schema, fixture repo clone).
- **Function-scoped** for everything else.
- **Never share mutable state across tests.** If a test mutates a fixture, the fixture is `function`-scoped.
- **Factories over bare data:** `make_run(status="failed")` beats a 30-line dict literal.

### 3.5 What gets unit-tested vs integration-tested

| Concern | Tier |
|---|---|
| Sonar issue ranking algorithm | Unit |
| Patch validator gates (each individually) | Unit |
| Pydantic schema validation | Unit |
| `propose_fix` prompt assembly | Unit (mock gateway) |
| LangGraph subgraph end-to-end against a fixture | Integration |
| API endpoint with auth + DB | Integration |
| Worker dequeues run, runs subgraph, publishes events | Integration |
| Login → connect repo → run → PR opens | E2E |

### 3.6 Mocking external services

- **LLM:** `respx` with recorded responses in `tests/fixtures/llm_recordings/`. Re-record only when the prompt or schema changes; commit the diff.
- **GitHub:** `respx`. Hand-crafted responses for specific status codes; record happy paths.
- **SonarQube:** never mocked at the HTTP level. We test the parser against fixture JSON; live API integration is exercised in a single dedicated integration test gated by an env flag.

### 3.7 What we don't test

- **Third-party libraries.** Don't test that `pydantic` validates correctly. Test our validators.
- **Trivial getters / DTOs.** Pydantic enforces them.
- **Logging output strings.** We test that the structlog redactor strips secrets; we don't assert on log message text.
- **Implementation details.** Test the contract, not the steps.

### 3.8 Async tests

- Use `pytest.mark.asyncio` (mode `strict` rejects unmarked async tests).
- Avoid `asyncio.sleep` in tests — use `asyncio.wait_for` with a tight timeout.
- For SSE / pub-sub tests, drain the stream into a list with a timeout; assert on the list.

---

## 4. Frontend testing (Angular)

### 4.1 Tooling

- **`ng test` (Karma + Jasmine)** for unit / component tests.
- **`@testing-library/angular`** for component tests — query by role/label, not by class.
- **`@playwright/test`** for E2E.
- **`axe-core`** wired into component tests for new views.
- **`msw`** for service-level HTTP mocking.

### 4.2 Component tests

- **Behavior over structure.** Don't assert on internal state; assert on what the user sees / can do.
- **`data-testid="..."`** attributes for selectors. Never `class` selectors in test queries.
- **No `fakeAsync` unless necessary.** Prefer `fixture.whenStable()` and signal `set()`.

```ts
it('disables submit while validating PAT', async () => {
  const { container, getByLabelText, getByRole } = await render(ConnectRepoComponent);
  await userEvent.type(getByLabelText(/Personal Access Token/i), 'ghp_test');
  await userEvent.click(getByRole('button', { name: /Validate/i }));
  expect(getByRole('button', { name: /Validate/i })).toBeDisabled();
});
```

### 4.3 E2E (Playwright)

E2E coverage is intentionally small. Run only the critical paths:

1. Login → repos list.
2. Connect repo (happy + invalid PAT).
3. New run → live progress → terminal redirect.
4. Run summary view + open PR link.
5. Cancel a running run.

E2E suite must complete in **<5 minutes** wall-clock.

---

## 5. Test data

### 5.1 Fixture repositories

- **Small, deterministic.** ~50 files max. Real syntax in target language; no random generation.
- **Committed, not generated.** Reproducibility is the priority.
- **Per-language sets:** `fixtures/repos/sample-java`, `sample-python`, `sample-rust`.

### 5.2 Fixture Sonar reports

- Hand-curated to cover the rule mix we care about.
- One "happy" report (all fixable), one "mixed" (some fixable, some skipped), one "pathological" (all hard cases).

### 5.3 LLM replay recordings

- Recorded once per scenario via `respx`'s record mode.
- Re-recorded when prompts change or model versions bump.
- Diffs reviewed in PR — large diffs trigger a "did the prompt regress?" question.

---

## 6. Flaky tests — zero tolerance

A test that fails intermittently is broken. We do **not** retry-to-pass.

If you discover a flaky test:
1. Quarantine it via `@pytest.mark.skip(reason="flaky #<issue>")` or `xit(...)` in Jasmine.
2. Open an issue tagged `flaky-test`.
3. Fix it within one week or delete it.

---

## 7. Performance tests

Not yet — added in M9 hardening. Until then:

- LLM latency tracked via the gateway's `latency_ms` field, surfaced in run summaries.
- API endpoint latency surfaced via Prometheus histograms.
- DB query timings logged at WARN when > 200ms.

---

## 8. Security tests

- **Test the secret redactor:** assert that a synthetic `ghp_xxx` token in a log call is replaced with `***`.
- **Test the workspace jail:** `FSTool.write("../../etc/passwd", ...)` raises.
- **Test the default-branch guard:** `GitHubTool.push(branch="main")` raises unless the override is set.
- **Test the auth dependency:** unauth'd request to a protected endpoint returns 401.
- **Test PAT decryption refusal in API process:** importing `secret_box.decrypt` from API code raises at import time.

These tests live in `tests/unit/security/` and never get skipped or quarantined.

---

## 9. CI integration

| Stage | What runs | When |
|---|---|---|
| `lint` | ruff, mypy, eslint, prettier | Every push |
| `test-unit-py` | `pytest tests/unit` | Every push |
| `test-unit-web` | `ng test --watch=false` | Every push |
| `test-integration` | `pytest tests/integration` against ephemeral compose | Every push |
| `test-e2e` | Playwright against booted stack | PR + nightly |
| `coverage-gate` | Reports + threshold check | After tests |
| `secret-scan` | gitleaks | Every push |
| `image-build` | Docker buildx for api, worker, web-ui | PR + main |
| `image-scan` | trivy | After build |

PRs cannot merge with any required check failing.

---

## 10. Local test loop — keep it fast

- `pytest tests/unit -q` — under 30s on a clean checkout.
- `pytest tests/unit/graph -q -x` — single area, fail-fast.
- `cd web && npm test -- --watch` — TDD mode.
- Mark slow integration tests `@pytest.mark.slow`; default invocation skips them. Run nightly + on PR.
