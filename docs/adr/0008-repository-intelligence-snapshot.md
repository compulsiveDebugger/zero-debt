# ADR-0008: Repository Intelligence is a Frozen Per-Run Snapshot

**Status:** Accepted

## Context

Every phase needs to know the target repo's structure: languages, frameworks, build systems, entry points, module graph. Two design shapes for this intelligence layer:

1. **Live index:** a service or in-process component that re-reads the working tree on demand, always returning the current state.
2. **Frozen snapshot:** build the outline once at run start, store it in state, hand it to every phase as read-only data.

A live index sounds more flexible but creates three problems:

- State is no longer JSON-serializable, breaking LangGraph checkpointing and resumability.
- Phase 1 mutates the working tree as it applies fixes; a live index would return a moving target, making phase logic depend on timing.
- Testing becomes harder — every test needs a live indexer rather than a fixture dict.

## Decision

The Repository Intelligence Layer produces a **frozen snapshot** (`repo_outline`, a plain dict) at `build_repo_outline` time. All phases read from this snapshot and treat it as read-only.

When a phase mutates the working tree (Phase 1 applying fixes, Phase 2 adding test files), the snapshot becomes stale for post-mutation questions. This is accepted: phases work against the outline as it was at run start, because the fixes they're generating reference that original structure.

If a phase genuinely requires a fresh outline mid-run, it must explicitly call `rebuild_outline()` which overwrites `repo_outline` — a deliberate, visible operation, not an implicit refresh. Phase 1 does not do this.

## Consequences

**Positive**

- Full JSON serialization → LangGraph checkpointer works end-to-end.
- Deterministic phase behavior — the outline a phase sees is fixed.
- Trivially testable — a fixture dict is enough.
- Simple mental model — "outline = view of repo as of clone time."

**Negative**

- Phases cannot discover files created by prior nodes within the same run. For P1 this is fine because new files aren't expected. For P2 (test generation) and P5 (Playwright scaffolds), generated files live in well-known paths that don't need indexing to be discoverable.
- `rebuild_outline()` is available but discouraged — cost is non-trivial for large repos.

**Discipline required**

- No phase reaches around the snapshot to read the working tree for structure discovery. If a phase thinks it needs live intelligence, escalate to a design review; don't just glob the filesystem.
