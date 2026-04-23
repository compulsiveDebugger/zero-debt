# ADR-0010: Never Write to Default Branches by Default

**Status:** Accepted

## Context

zero-debt is an autonomous agent that commits and pushes code. The worst-case failure mode is a bad commit landing on `main` or `master` — it's visible, harder to revert cleanly once CI runs, and erodes user trust in the tool immediately.

Even with well-tested code, defense-in-depth matters here because:

- Bugs in branch-name derivation could point commits at `main`.
- A misconfigured token with push access to default branches makes this possible.
- A user could pass `--base-branch main --target-branch main` by mistake.

## Decision

The `GitHubTool` and `GitTool` refuse to target a repository's default branch (or any branch literally named `main` / `master` as a safety fallback) for mutating operations (push, force-push, merge, direct commit) **unless** the operator has explicitly set `config.github.allow_default_branch=true`.

Additional guards:

- Work-branch names are always prefixed `zero-debt/<phase>-<runid>` — this is enforced in `commit_and_push_branch` and asserted in a unit test.
- A run whose `work_branch` equals the default branch immediately fails fast with a structured error, regardless of tool-layer checks.
- PR creation always targets default branch **as the base** and our work branch **as the head** — never the inverse.

## Consequences

**Positive**

- Single-point policy enforcement at the tool layer. No phase can bypass it by accident.
- `allow_default_branch=true` is conspicuous in config review — any PR that flips it raises a reviewer's eyebrow.
- Defense in depth — multiple layers would all have to fail simultaneously for an accidental default-branch write.

**Negative**

- There is a legitimate edge case (e.g., a doc-only agent fixing a typo in `main` on a personal project) that this blocks by default. Users with that need set the flag explicitly. Acceptable friction given the blast radius of the failure mode.

**Non-negotiable**

- This default does not get loosened "for convenience" in v1. If real usage patterns justify a more nuanced policy (e.g., allow for docs-only paths), that's a future ADR — not a quiet default change.
