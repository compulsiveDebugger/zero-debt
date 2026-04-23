# Contributing to zero-debt

Welcome. This is the entry point for anyone making changes to the codebase. **Read this once, then keep [docs/ONBOARDING.md](docs/ONBOARDING.md) open while you set up.**

zero-debt is built in **Python 3.11+** (backend) and **Angular 20 + TypeScript** (frontend). Java / Rust / TypeScript appear as *target languages* the validator must understand — see [docs/LANGUAGE_TARGETS.md](docs/LANGUAGE_TARGETS.md).

---

## Before you contribute

1. **Read** [README.md](README.md), [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), and the relevant ADRs in [docs/adr/](docs/adr/).
2. **Set up your environment** via [docs/ONBOARDING.md](docs/ONBOARDING.md).
3. **Open an issue** before any non-trivial change. We'd rather discuss design once than redo work.
4. **Check scope** in [docs/03_scope_timeline_risks.md](docs/03_scope_timeline_risks.md) §1.3 — out-of-scope changes need a scope amendment.

---

## How to propose a change

| Change type | Process |
|---|---|
| Bug fix | Issue → branch → PR. Include a regression test. |
| Small feature | Issue → discussion comment → branch → PR. |
| New phase, new tool, new datastore, breaking API change | RFC issue → architectural review → ADR → branch → PR. **Do not start coding without ADR consensus.** |
| New external dependency | Follow [docs/DEPENDENCY_POLICY.md](docs/DEPENDENCY_POLICY.md). |
| Documentation only | Branch → PR. Tag with `docs` label. |

---

## Branching and commits

Full convention in [docs/GIT_WORKFLOW.md](docs/GIT_WORKFLOW.md). Quick version:

- Branch from `main`. Naming: `<type>/<short-kebab-summary>` where `type ∈ {feat, fix, docs, chore, refactor, test, perf, security}`. Example: `feat/sse-reconnect-banner`.
- Commits: **Conventional Commits** (`feat(api): add SSE reconnect handler`).
- One logical change per commit. No "wip" commits in `main`.
- Never force-push `main`. Force-push your own branch only.

---

## Code style — non-negotiable

| Stack | Tooling | Detail |
|---|---|---|
| Python | `ruff` (lint + format), `mypy --strict` | [docs/CODE_STYLE_PYTHON.md](docs/CODE_STYLE_PYTHON.md) |
| Angular / TS | `eslint`, `prettier`, `tsc --noEmit` strict | [docs/CODE_STYLE_ANGULAR.md](docs/CODE_STYLE_ANGULAR.md) |
| API design | OpenAPI-driven; envelopes per [docs/API_STYLE.md](docs/API_STYLE.md) |
| Logging | structlog JSON; key conventions per [docs/LOGGING_STYLE.md](docs/LOGGING_STYLE.md) |
| Tests | Per [docs/TESTING.md](docs/TESTING.md) — coverage gates, fixture rules, no flakies |
| Security | Per [SECURITY.md](SECURITY.md) — no secrets in YAML, no shell=True, no LLM self-eval gates |

A PR that fails lint, type-check, or any test is not reviewed. Fix it first.

---

## Pull request expectations

Use the [PR template](.github/PULL_REQUEST_TEMPLATE.md). A reviewable PR:

- **Stays under 400 LoC of diff** where possible. Larger? Split it.
- **Updates docs in the same PR** if it changes behavior covered by docs.
- **Includes tests** for new behavior. Bug fixes include a regression test.
- **Updates [CHANGELOG.md](CHANGELOG.md)** under `## Unreleased`.
- **Adds or updates an ADR** if it changes a load-bearing contract (see [docs/adr/README.md](docs/adr/README.md) §"When to add a new ADR").
- **Has a clear title** following Conventional Commits.
- **Passes CI** — required checks block merge.

A PR that touches:
- Core orchestrator (`graph/orchestrator.py`, `graph/routing.py`, state schema) — **requires two reviewers**, one being a maintainer.
- Auth, secrets, or PR/branch creation logic — **requires a security-tagged reviewer**.
- A migration — **requires a database-tagged reviewer** and must be tested via `alembic upgrade head && alembic downgrade -1`.

---

## Reviews

You are responsible for getting your PR reviewed. As a reviewer:

- Be specific. "This is wrong" is not a review.
- Distinguish blockers from suggestions. Use **`blocking:`** prefix for blockers.
- Approve only when you'd be comfortable maintaining the code yourself.

---

## Merge

- **Squash and merge** for feature branches → `main`. Subject must follow Conventional Commits.
- **Rebase and merge** for stacked PRs (rare; coordinate first).
- Never **merge commit**.

CI must be green. CHANGELOG must reflect the change. Then merge.

---

## After merge

- Delete the branch (the GitHub UI does this automatically when configured).
- If the change ships in a release, the release process ([docs/RELEASING.md](docs/RELEASING.md)) updates the CHANGELOG and tags.

---

## Reporting issues

- **Bugs:** [.github/ISSUE_TEMPLATE/bug_report.md](.github/ISSUE_TEMPLATE/bug_report.md). Include reproduction steps and expected vs. actual.
- **Feature requests:** [.github/ISSUE_TEMPLATE/feature_request.md](.github/ISSUE_TEMPLATE/feature_request.md). State the user need, not the solution.
- **Security vulnerabilities:** **Do not open a public issue.** Follow [SECURITY.md](SECURITY.md).

---

## Conduct

By participating you agree to abide by [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md). Be direct, be kind, leave the codebase better than you found it.

---

## Quick links

- Architecture: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- Phase 1 plan: [docs/PHASE-1-BLUEPRINT.md](docs/PHASE-1-BLUEPRINT.md)
- Implementation plan: [docs/02_implementation_plan.md](docs/02_implementation_plan.md)
- Roadmap: [docs/ROADMAP.md](docs/ROADMAP.md)
- ADRs: [docs/adr/](docs/adr/)
- Glossary: [docs/GLOSSARY.md](docs/GLOSSARY.md)
