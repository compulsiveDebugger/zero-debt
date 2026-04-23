# Dependency Policy

Adding a dependency creates a forever-cost: security scanning, version pinning, license review, supply-chain risk, image bloat. We're stingy on purpose.

---

## 1. Before adding a new dependency

Ask, in order:

1. **Can the standard library do it?** Python's stdlib and modern TypeScript are both deep. Try first.
2. **Does an existing dependency already cover it?** We bring in `httpx` already — don't add `requests`. We have `pydantic` — don't add `marshmallow`.
3. **Is the dep maintained?** Last release < 12 months. Open-issue triage active. Not a one-person hobby project for production-critical paths.
4. **What's the install cost?** Image size, transitive count, native build deps. Big native deps need a stronger justification.
5. **What's the security posture?** Track record, response time on CVEs.
6. **What's the license?** See §4.

If you've answered all six and the answer is still "yes, add it," open a PR. The PR description **must include**:

```
### New dependency: <name> <version>

- **Why:** <one paragraph>
- **Considered alternatives:** <list>
- **Maintenance:** <last release, active issues, maintainer count>
- **Image impact:** <MB added>
- **License:** <SPDX id>
- **Security:** <recent CVEs, response cadence>
```

---

## 2. Version pinning

| Stack | Style | Rationale |
|---|---|---|
| Python | `name>=X.Y,<Z.0` upper-bounded to next major | Prevents breaking minor pulls in CI; explicit major bumps |
| Node | `"name": "^X.Y.Z"` (caret) | Standard npm semantics |

We do **not** pin to exact versions in source (`pyproject.toml`, `package.json`). The lockfile (`uv.lock`, `package-lock.json`) is the exact-pin authority and **is committed**.

---

## 3. Update cadence

- **Renovate / Dependabot** opens PRs for every dep update.
- **Patch + minor updates:** auto-merged after CI passes if the dep is already an approved dependency.
- **Major updates:** human review. Read the changelog. Test the affected paths.
- **Security updates:** prioritized — handle within 1 business day.

---

## 4. License policy

| License | Allowed? | Notes |
|---|---|---|
| MIT, BSD (2/3-clause), Apache 2.0, ISC | ✅ Yes | Default |
| MPL 2.0, LGPL | ⚠️ Yes for libraries; review for static linking implications | Rare |
| GPL (any version) | ❌ No | Copyleft — viral if linked |
| AGPL | ❌ No | Network copyleft |
| SSPL, BSL, Elastic License v2 | ❌ No | Source-available, not OSS |
| Unlicensed / custom | ❌ No | Treat as proprietary |
| WTFPL, "do whatever" | ⚠️ Avoid | Legally murky |

CI runs `pip-licenses` and `license-checker` and fails on disallowed licenses.

---

## 5. Approved-list management

`pyproject.toml` and `package.json` are the approved lists. Anything not listed there is unapproved by definition. Transitive deps are subject to the same license policy but not the same justification process.

If a transitive dep introduces a problematic license or a critical CVE: pin it explicitly to a different version, or replace the parent dep.

---

## 6. Vendoring

We do not vendor dependencies. Two narrow exceptions:

1. **A small snippet** (≤30 lines) we want to modify or that has unclear maintenance. Copy with attribution into `src/zero_debt/_vendor/<name>.py`. License must be MIT/BSD/Apache 2.0.
2. **A pinned generated client** (e.g., the OpenAPI Angular client). Lives in `web/src/app/core/api/generated/`, regenerated via tooling. CI checks it's in sync.

---

## 7. Removing a dependency

When a feature dies, the dep often dies with it. As part of the cleanup PR:

- Remove from `pyproject.toml` / `package.json`.
- Regenerate the lockfile.
- Search for stragglers: `grep -r "import <name>"`.
- Note the removal in CHANGELOG under `Removed`.

---

## 8. Security scanning

| Stage | Tool |
|---|---|
| Pre-commit | `gitleaks` (secrets only) |
| CI per push | `pip-audit`, `npm audit --audit-level=high` |
| CI per build | `trivy image` on each container image |
| Renovate / Dependabot | Continuous CVE-driven PRs |

A high-or-critical finding from any scanner blocks merge.

---

## 9. Forks and patches

If we discover a bug in a dep:

1. Open an issue upstream.
2. If we can't wait, fork to `<corp-org>/<dep>` and pin to our fork's commit SHA in `pyproject.toml`.
3. Continue pushing fixes upstream — we don't want to maintain a permanent fork.
4. As soon as upstream merges, switch back.

---

## 10. AI / model providers — special case

LLM SDKs (`anthropic`, `openai`) deserve extra scrutiny:

- **Pin to minor version range.** `anthropic>=0.40,<1.0`.
- **Update separately from anything else.** Bumping the SDK can change response shapes.
- **CI runs the LLM contract tests** against recorded fixtures on every SDK PR.
- **Never auto-merge.** Always human review.

---

## 11. The "dependency floor"

These deps are load-bearing and cannot be removed without an ADR:

- `langgraph`, `langchain-core` (orchestration)
- `pydantic` (every data contract)
- `fastapi`, `uvicorn` (API tier)
- `arq`, `redis` (queue)
- `psycopg`, `sqlalchemy`, `alembic` (DB)
- `motor` (Mongo)
- `anthropic` (LLM)
- `argon2-cffi` (auth)
- `structlog` (logging)
- `@angular/core`, `rxjs`, `tailwindcss` (frontend)

Anything else can be reconsidered.
