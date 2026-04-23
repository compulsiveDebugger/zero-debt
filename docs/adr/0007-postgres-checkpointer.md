# ADR-0007: Postgres-Backed LangGraph Checkpointer in Production

**Status:** Accepted

## Context

LangGraph supports in-memory, SQLite, and Postgres checkpointer backends. zero-debt runs can take tens of minutes, traverse many external API calls, and must survive container restarts without redoing LLM work (which is expensive and nondeterministic).

Backend characteristics:

- **In-memory** — zero persistence; fine for tests, useless for prod.
- **SQLite** — persistence, but single-writer and file-locked; adequate for dev but doesn't scale to concurrent runs and doesn't isolate well in containerized deployments where the volume is shared.
- **Postgres** — robust, concurrent, network-addressable, standard ops story.

## Decision

- **Production / containerized dev:** Postgres 16 via the `checkpoint-db` Compose service. Configured by `POSTGRES_URL`.
- **Unit tests:** in-memory checkpointer.
- **Local ad-hoc dev (optional):** SQLite via a file path in `POSTGRES_URL` replaced with a SQLite DSN. Selected by the `run.checkpointer` YAML key.

Checkpoint boundaries are declared explicitly at nodes that:
- Mutate the working tree (`apply_patch`).
- Call external APIs (`propose_fix`, any GitHub mutation).
- Complete a subgraph (on subgraph exit).

A background TTL sweep (planned for post-M6) archives completed-run checkpoints after 30 days to a separate table to cap working-set size.

## Consequences

**Positive**

- A failed run resumes from the last checkpoint — never redoes LLM calls needlessly.
- Concurrent runs (multiple phase invocations against different repos) don't fight over a file lock.
- Standard Postgres ops tooling applies: backups, migrations, monitoring.
- Clean separation between dev (SQLite) and prod (Postgres) via a config switch, not code.

**Negative**

- Another service to operate. Justified because we already need persistent state and Postgres is the industry baseline.
- Schema migrations tied to LangGraph version upgrades. Pinned versions + CI migration checks mitigate.
- Sensitive data (LLM prompts if `include_llm_prompts=true`) lands in Postgres — must be treated like any secret-containing database.

**Discipline required**

- Every node that crosses an external boundary should be positioned so LangGraph treats it as a checkpoint. Framework does this automatically for async boundaries; resist the urge to merge a checkpointable node with a non-checkpointable one.
