# ADR-0017: Vector Store Abstraction — pgvector Default, Dedicated Stores Deferred

**Status:** Accepted

## Context

Several phases benefit from semantic code retrieval:

- **P3 (Code Reviewer):** "find similar patterns across the repo" to produce consistent review feedback.
- **P4 (PR Auto-Fixer):** retrieve rationale from prior similar fixes.
- **P5 (E2E Generator):** match UI routes/flows against existing test fixtures.

P1 does **not** need a semantic index — its context comes from the structural RIL outline and the Sonar-report-targeted file slice.

Options for the vector backend:

1. **Dedicated vector DB** — Qdrant, Weaviate, Milvus. Best performance at scale; another service to operate.
2. **pgvector extension** on the existing Postgres. Good enough through millions of vectors; zero new services.
3. **Embedded** — FAISS, ChromaDB in-process. Simple but doesn't fit a multi-worker architecture where any worker might serve any run.

## Decision

Two-layer decision:

1. **Abstract behind an interface.** All phase code talks to `VectorStore`, never to a concrete backend:
    ```python
    class VectorRecord(BaseModel):
        id:        str
        embedding: list[float]
        metadata:  dict     # repo_connection_id, file_path, chunk_id, language, ...

    class VectorStore(ABC):
        @abstractmethod
        async def upsert(self, records: list[VectorRecord]) -> None: ...
        @abstractmethod
        async def query(
            self,
            embedding: list[float],
            k: int,
            filter: Optional[dict] = None,
        ) -> list[VectorRecord]: ...
        @abstractmethod
        async def delete_repo(self, repo_connection_id: str) -> None: ...
    ```

2. **Use pgvector as the v1 backend.** It rides on the existing Postgres; the image is `pgvector/pgvector:pg16`. `ivfflat` index on the `embedding` column; filter-before-search via standard SQL `WHERE`.

Backend selection is a single config knob: `vector_store.backend = pgvector | qdrant | weaviate`. Qdrant/Weaviate backends remain stubs until real scale or latency pressure justifies building them.

Embedding provider lives behind `LLMGateway` — either OpenAI's `text-embedding-3-small` (configurable) or a local `sentence-transformers` model for offline deployments.

## Consequences

**Positive**

- One fewer service in the demo/early-production stack.
- Phase code is backend-agnostic from day one — migration later is a config flip plus operational runbook, not a refactor.
- Transactional guarantees between repo-connection rows and their vectors (same database).
- Filter-then-search is trivial with SQL.

**Negative**

- pgvector performance tapers well below dedicated vector DBs at extreme scale (hundreds of millions of vectors). We're nowhere close.
- Vector writes add write load to the operational Postgres. Mitigated by running embedding pipelines off the request hot path.

**Non-negotiable**

- Phase code **never** issues a raw SQL query against the vector table. Always through `VectorStore`. Otherwise the abstraction leaks and future migration becomes a refactor.
