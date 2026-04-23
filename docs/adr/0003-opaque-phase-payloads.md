# ADR-0003: `phase_input` / `phase_output` are Opaque Dicts in Shared State

**Status:** Accepted

## Context

The shared `ZeroDebtState` TypedDict travels through every node in the orchestrator. If we strongly type every phase's inputs and outputs at the orchestrator level, adding Phase 2 requires modifying `ZeroDebtState` → which forces a TypedDict migration and touches every node that reads shared state.

This is the classic "shared-state pollution" trap. We need per-phase type safety **without** coupling the orchestrator to phase-specific schemas.

## Decision

`phase_input` and `phase_output` are typed as `Dict[str, Any]` in `ZeroDebtState`. Each phase subgraph owns its own Pydantic contract at its boundary:

- On entry: the subgraph parses `state["phase_input"]` via its own `PhaseInputModel.model_validate(...)`.
- On exit: the subgraph serializes its output with `model.model_dump(mode="json")` and writes it back to `state["phase_output"]`.

The orchestrator never inspects the contents of these dicts beyond passing them through. Post-phase normalization (`post_phase_normalize`) adapts them to the delivery contract using a per-phase adapter registered at phase-module import time.

## Consequences

**Positive**

- Orchestrator and shared state schema are stable across all five phases — zero migration burden when P2–P5 land.
- Per-phase contract evolution stays local to that phase's module.
- JSON-safe by construction, which is what LangGraph's checkpointer wants.

**Negative**

- No compile-time type safety on `phase_input` / `phase_output` at the orchestrator boundary. Mitigated by Pydantic validation at the subgraph entry — errors surface immediately, not silently.
- Introspection tools (e.g., state snapshots in a debug UI) see `Dict[str, Any]` instead of structured types. Mitigated by having each phase register its contract in a registry that debugging tools can consult.

**Discipline required**

- Each phase **must** validate its `phase_input` on entry. Skipping validation to "save a step" is how this decision becomes a bug generator.
- The opaque dict is a boundary, not a dumping ground. Internal subgraph working fields live under `phase_input.internal_*` namespacing; public fields are first-class keys.
