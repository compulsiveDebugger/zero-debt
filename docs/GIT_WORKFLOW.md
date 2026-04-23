# Git Workflow

Branching, commits, PRs, releases. Every contributor follows this. CI enforces what it can.

---

## Branching model

Trunk-based. Single long-lived branch: `main`. All work happens on short-lived feature branches.

- `main` is always **deployable** to the demo environment.
- A feature branch is **<1 week old** at merge. Older = re-evaluate scope.
- No `develop`, `release/*`, or `hotfix/*` branches. Hotfixes go to `main` and get cherry-picked into a release tag if needed.

### Branch naming

`<type>/<short-kebab-summary>[-<issue-number>]`

| Type | Use for |
|---|---|
| `feat` | New user-visible behavior |
| `fix` | Bug fix |
| `refactor` | No behavior change, structural improvement |
| `perf` | Performance improvement (with measurement) |
| `test` | Test-only change |
| `docs` | Documentation-only |
| `chore` | Tooling, deps, build, CI |
| `security` | Security-relevant change (still public; report private vulns per SECURITY.md) |

Examples:
- `feat/sse-reconnect-banner-321`
- `fix/pat-decryption-leak`
- `chore/bump-langgraph-0-3`

---

## Commit conventions — Conventional Commits

Format: `<type>(<scope>): <subject>`

- `<type>` ∈ same set as branch types.
- `<scope>` is optional but encouraged. Common scopes: `api`, `worker`, `graph`, `ui`, `auth`, `db`, `llm`, `phase1`, `infra`, `docs`.
- `<subject>` is **imperative, lowercase, no period**, ≤72 chars.
- Body (optional) explains **why**. Wrap at 80.
- Footer (optional): `Refs: #123`, `BREAKING CHANGE: <description>`, `Co-authored-by: ...`.

**Good:**
```
feat(graph): add cooperative cancel check between nodes

Workers now poll a Redis cancel flag at every node boundary. The check is
cheap (single GET) and unblocks UI Cancel within one node duration instead
of waiting for the run hard timeout.

Refs: #284
```

**Bad:**
```
fixed the bug                       # which bug? what fix?
WIP                                 # never in main
Fixed cancellation. Also reformatted everything.   # mixed concerns
```

A pre-commit hook validates commit messages.

---

## One commit = one logical change

If you can't summarize the change in a 72-char subject, it's two changes. Split.

- Feature work + drive-by formatting? Two commits.
- Test for new code lives in the **same commit** as the code.
- Migration + the model change that needs it = one commit.

---

## Pull requests

### Opening

1. Push your branch.
2. `gh pr create` (or use the UI). Use the PR template — answer every section.
3. Mark **draft** if not ready for review. Mark **ready for review** when CI is green.
4. Assign reviewers. Don't wait to be picked up.

### PR title

Same Conventional Commits format as commits — this becomes the squash-merge subject.

### Size

Target **<400 LoC of diff**, excluding generated files. Larger:
- Split into stacked PRs (each one separately reviewable and mergeable).
- If truly atomic and you can't split: warn reviewers in the PR description and break into commits that make a logical reading order.

### Reviews

- **At least one approval** required.
- **Two approvals** required if the PR touches: orchestrator core, auth, secret handling, branch/PR creation, migrations, ADRs.
- Use **`blocking:`** prefix on comments that must be addressed before merge. Other comments are suggestions.
- Reviewer SLA: first response within **1 business day**. If you can't, decline the request so it can be reassigned.
- Author SLA: respond to all blocking comments within **2 business days** or close the PR.

### Merging

- Required checks must be green: lint, type-check, unit tests, integration tests, container build, secret scan.
- **Squash and merge.** Subject must be Conventional Commits. Squash body should preserve commit-by-commit context only if useful; otherwise leave just the PR description.
- After merge, the GitHub UI deletes the branch.

### Merging stacked PRs

If you have PR #2 stacked on PR #1:
- Merge #1 first. GitHub auto-rebases #2 onto `main`.
- If conflicts arise: rebase #2 manually, force-push.

---

## Force pushes

- **Allowed** on your own feature branches.
- **Forbidden** on `main` (branch protection enforces).
- **Avoid** during active review — leave review comments anchored. Use `git commit --fixup` then `git rebase -i --autosquash` before final review.

---

## Tags & releases

Semantic versioning. Tag format: `v<major>.<minor>.<patch>`.

- **Patch** (`v0.1.1`): bug fixes, docs, no behavior change for callers.
- **Minor** (`v0.2.0`): new features, backwards-compatible.
- **Major** (`v1.0.0`): breaking changes — to API, config, schema, or contracts.

Release process: see [docs/RELEASING.md](RELEASING.md) (added in M9).

While pre-1.0, **minor bumps may include breaking changes** but must be called out in CHANGELOG under `BREAKING CHANGE:`.

---

## Special branches

- **`renovate/*` / `dependabot/*`** — automated dependency PRs. Handle these promptly; they're security-relevant.
- **`docs/*`** — docs-only PRs. Single-reviewer rule, faster turnaround.

---

## Anti-patterns to avoid

- **Long-lived feature branches.** Rebase or split before they age past a week.
- **"Fix review feedback" commits in the same PR after merge.** They hide context. Use `--fixup`.
- **Mixing unrelated changes.** Refactor + new feature + dep bump in one PR is unreviewable.
- **Bypassing CI.** If CI is broken, fix CI first; don't merge with `--no-verify`.
- **Force-pushing main.** Branch protection blocks this; if you somehow have permission, do not.

---

## When in doubt

Match the style of the most recent merged PRs in the area you're touching. They embody the current consensus.
