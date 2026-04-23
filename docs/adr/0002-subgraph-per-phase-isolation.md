# ADR-0002: One Independently-Compilable Subgraph per Phase

**Status:** Accepted

## Context

zero-debt will ship Phase 1 first, then Phases 2–5 incrementally over months. We need a topology that:

- Lets a single phase be developed, tested, and deployed without touching the others.
- Keeps the pre/post pipeline DRY — no copy-paste of clone/outline/push across five workflows.
- Avoids a god-graph where all five phases' nodes are flattened into one mega-StateGraph.

Two candidate structures:

1. **Flat graph:** one `StateGraph` with all nodes for all phases, gated by conditional edges on `active_phase`. Works but forces every test harness and every compile to know about every phase. Blast radius for a P5 refactor reaches into P1.
2. **Nested subgraphs:** each phase is a separate `StateGraph` compiled into a `CompiledStateGraph`, mounted as a single node in the parent graph via `phase_router`.

## Decision

Each phase is implemented as an **independently-compilable subgraph** exported from `graph/subgraphs/<phase>.py` as `compile_subgraph() -> CompiledStateGraph`. The parent graph's `phase_router` conditionally routes to whichever subgraph matches `active_phase`.

Subgraph integration rules:

- A subgraph reads from and writes to the shared `ZeroDebtState` but **must** funnel its public outputs through `phase_output` (an opaque dict — see [ADR-0003](0003-opaque-phase-payloads.md)).
- A subgraph owns its own internal state keys under `phase_input` but must not mutate orchestrator-owned fields (`status`, `errors`, `work_branch`, etc.) directly — those belong to post-phase nodes.
- A subgraph is unit-testable in isolation by invoking its compiled form with a minimal fixture state.

## Consequences

**Positive**

- P1 ships end-to-end without P2–P5 existing. P2–P5 add without touching `orchestrator.py`.
- Each subgraph has its own test suite, own prompt pack, own Pydantic contract — clean ownership boundaries.
- The parent graph stays small and readable.

**Negative**

- Slightly more boilerplate per phase (compile function, router case).
- Cross-phase state sharing is deliberately awkward — by design, but occasionally frustrating.

**Discipline required**

- The router is the **only** orchestrator change allowed when adding a phase. If a proposed phase forces other orchestrator edits, that's a signal the contract is being violated.
