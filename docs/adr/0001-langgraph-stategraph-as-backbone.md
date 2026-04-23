# ADR-0001: LangGraph `StateGraph` as Orchestration Backbone

**Status:** Accepted

## Context

zero-debt runs five multi-step agentic workflows that share clone/outline/deliver pre- and post-phases. Each workflow needs:

- Deterministic routing between steps with explicit conditional branches.
- Resumability after container restarts or API failures.
- Per-step observability (timings, errors, LLM usage).
- The ability to compose a workflow out of smaller reusable graphs.

Candidate orchestration approaches considered:

1. **Plain Python function chains + async primitives.** Fast to start, but state management, checkpointing, and conditional routing become ad-hoc infrastructure we'd have to build ourselves.
2. **Celery / Temporal / Airflow.** Heavy for what is ultimately a sub-minute to tens-of-minutes workflow. Operational cost too high for a single-tenant tool.
3. **LangGraph `StateGraph`.** Purpose-built for agentic workflows with state, conditional routing, subgraphs, and a pluggable persistence layer.

## Decision

Use LangGraph's `StateGraph` as the single orchestration primitive across zero-debt. No custom workflow engine. No plain Python function chains for multi-step work.

- Top-level graph wires pre-phase nodes → `phase_router` → phase subgraphs → post-phase nodes.
- Checkpointing is delegated to LangGraph's persistence layer (see [ADR-0007](0007-postgres-checkpointer.md)).
- Observability hooks (logging, tracing) wrap each node at the framework level, not inside node bodies.

## Consequences

**Positive**

- Free checkpointing and resumability — no bespoke state-machine code.
- Subgraphs (ADR-0002) drop out naturally from the framework.
- Conditional edges are first-class — no magic if/else chains buried in nodes.
- Battle-tested observability pattern (LangSmith / OpenTelemetry integration).

**Negative**

- Coupling to a still-evolving library. Version pins in `pyproject.toml` matter; upgrade cadence planned quarterly.
- Team must internalize LangGraph mental model (reducers, partial state updates). Investment amortizes across all five phases.
- Some debugging surfaces are framework-shaped, not Python-shaped — stack traces can be less direct.

**Mitigation for negative consequences**

- Pin `langgraph>=0.2.0,<1.0` and test upgrades in CI before bumping.
- Keep node bodies as plain async functions so they're unit-testable without the framework.
