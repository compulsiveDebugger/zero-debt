# Python Code Style

The backend (`api`, `worker`, LangGraph orchestrator, all tools) is Python 3.11+. This document is what reviewers will hold you to.

**Tooling enforces what it can:** `ruff` (lint + format), `mypy --strict`, `pytest`. Things tooling can't enforce are below.

---

## 1. Formatting (auto-enforced)

- **`ruff format`** — line length 100, double quotes, trailing commas in multi-line.
- **`ruff check --fix`** — combines lint + import sort. Rule set: `E,F,W,I,B,C4,UP,SIM,RET,RUF` minus our exceptions in `pyproject.toml`.
- Never disagree with `ruff` in a PR. If a rule is genuinely wrong for our context, propose an exception in `pyproject.toml` in a separate PR.

---

## 2. Type hints — strict

- **Every function signature has type hints** for parameters and return type.
- **No `Any` without a comment** explaining why. Reviewers grep for unjustified `Any`.
- **Use `from __future__ import annotations`** at the top of every module — postponed evaluation, cheaper imports, cleaner forward refs.
- **Prefer Pydantic v2 models** for any external boundary (API, LLM, DB, file I/O). Internal data structures can use `dataclass` or `TypedDict` when no validation is needed.
- **Use precise types:**
  - `list[str]` not `list`.
  - `Sequence[T]` for read-only inputs; `list[T]` for outputs you'll mutate.
  - `Mapping[K, V]` for read-only dict params.
  - `Path` not `str` for filesystem paths.
  - `UUID` not `str` for ids.
  - `datetime` always timezone-aware (UTC).

```python
from __future__ import annotations
from collections.abc import Sequence
from pathlib import Path
from uuid import UUID

async def load_issues(report_path: Path, allowed_rules: Sequence[str]) -> list[SonarIssue]:
    ...
```

---

## 3. Naming

| Kind | Convention | Example |
|---|---|---|
| Module | `snake_case` | `propose_fix.py` |
| Class | `PascalCase` | `SonarReport` |
| Function / variable | `snake_case` | `validate_patch` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_TOKEN_BUDGET` |
| Private | `_leading_underscore` | `_normalize_path` |
| Type alias | `PascalCase` | `RepoOutline = dict[str, Any]` |
| Enum value | `UPPER_SNAKE_CASE` | `RunStatus.PARTIAL` |

Avoid:
- Single-letter names except for trivially scoped iterators (`i`, `n`).
- Abbreviations (`cfg`, `mgr`, `svc`) — write the word.
- `data`, `info`, `result`, `obj` — use the actual concept.

---

## 4. Async-first

- **Every I/O-bound function is `async def`** — DB, HTTP, file I/O, Redis, Mongo.
- **CPU-bound work in `async` runs the loop on a thread:** use `await asyncio.to_thread(fn, ...)`.
- **No mixing sync and async clients** for the same resource. We use:
  - `psycopg[async]` not `psycopg2`
  - `redis.asyncio` not `redis`
  - `motor` not `pymongo`
  - `httpx.AsyncClient` not `requests`
- **Bound concurrency** with `asyncio.Semaphore` when fanning out — never unbounded `gather` over user input.

---

## 5. Functions

- **Single responsibility.** If you can describe what it does with "and", split.
- **Keep ≤50 lines.** Longer = restructure.
- **Keep ≤4 positional params.** More = pass a typed dataclass / Pydantic model.
- **Pure where possible.** Side effects (logging, DB writes) are explicit, not hidden in a helper named `format_x`.

---

## 6. Classes

- Prefer **functions and modules** over classes. Reach for a class when state and behavior genuinely belong together.
- **No god classes.** Each LangGraph node is a function or a small focused class — see [ADR-0001](adr/0001-langgraph-stategraph-as-backbone.md).
- **`@dataclass(frozen=True, slots=True)`** for value objects.
- **`Protocol` over inheritance** when defining what a tool must look like.
- **Avoid mixins.** Composition over inheritance.

---

## 7. Errors

- **Define a small set of explicit exception types** per domain — `LLMSchemaError`, `PatchValidationError`, `WorkspaceJailViolation`. Never raise `Exception` or `RuntimeError` from production code.
- **Catch the narrowest exception** that makes sense. `except Exception` only at top-level handlers.
- **Re-raise with context.** `raise NewError(...) from original_error`.
- **Errors have structured payloads** when they cross a boundary (API response, log line, run state). Pydantic models help.
- **No silent excepts.** Every `except` either re-raises, logs at WARN+ with context, or has a comment explaining why swallowing is correct.

```python
try:
    return await self._gateway.complete(req)
except LLMRateLimited as e:
    log.warning("llm_rate_limited", retry_after=e.retry_after, request_id=req.request_id)
    raise
except LLMServerError as e:
    raise LLMRetriableError(f"upstream {e.status}") from e
```

---

## 8. Logging

Full conventions in [LOGGING_STYLE.md](LOGGING_STYLE.md). Quick rules:

- **Always `structlog`**, never stdlib `logging` directly.
- **Event name first, kwargs for context.** `log.info("patch_applied", run_id=..., file=..., issues_closed=2)`.
- **Lowercase snake_case event names.** No string interpolation in the event name.
- **No PII / secrets.** The redactor catches the obvious; you're still responsible.
- **Correlation:** bind `run_id`, `user_id`, `phase` once at task start; they propagate via contextvars.

---

## 9. Imports

- **Absolute imports only** within `src/zero_debt/`. No relative imports.
- **Standard library → third-party → local**, separated by blank lines. `ruff` enforces this.
- **No `from x import *`** anywhere.
- **No top-level side effects** in modules. Only definitions.

---

## 10. Tests

Full guide in [TESTING.md](TESTING.md). Style highlights:

- **One assertion concept per test.** Multiple `assert` lines are fine if they describe the same behavior.
- **Arrange-Act-Assert** layout. Blank lines between sections.
- **Descriptive names:** `test_validate_patch_rejects_overlap_miss`, not `test_validate_patch_3`.
- **Fixtures over setup methods.** No `unittest.TestCase`; pure pytest.
- **`@pytest.mark.asyncio`** for async tests; `pytest-asyncio` mode `strict`.

---

## 11. Comments and docstrings

- **Default to no comments.** Names should carry the meaning.
- **Write a comment when the WHY is non-obvious** — a workaround, a perf hack, a rule from an external system.
- **Module-level docstring** when the module's purpose isn't obvious from its name. One paragraph max.
- **Public API functions** in `tools/`, `llm/`, `auth/` get a one-line docstring describing the contract. Skip args/returns sections — types carry that.
- **Never document WHAT** — well-named code carries that. Never reference the current task or PR — those rot.

---

## 12. Configuration access

- **All config goes through `get_settings()`** — the cached Pydantic `BaseSettings` instance.
- **Never `os.environ.get(...)` in business logic.** Add a typed field to `Settings` instead.
- **Never read a YAML file directly** outside `settings.py`.

---

## 13. Database & external clients

- **Always inject** the `AsyncSession` / Redis / Mongo / `LLMGateway` into your function as a parameter. Never construct one inside business logic.
- **FastAPI `Depends`** for the API tier; explicit kwargs for everything else.
- **Don't reach into `tool.private_attr`** — the Tool layer is the contract.

---

## 14. Concurrency safety

- **No module-level mutable state.** No singletons that hold per-request data.
- **Contextvars** for per-task state (e.g., `current_run_id`).
- **No `threading.Lock`** in async code. Use `asyncio.Lock`.
- **Never block the event loop.** No `time.sleep`, no sync `requests`, no sync DB driver call.

---

## 15. Performance

- **Measure before optimizing.** Profile with `cProfile` or `pyinstrument`; never guess.
- **Bound everything.** No unbounded queues, no unbounded retries, no unbounded fan-out.
- **JSON serialization:** prefer Pydantic `model_dump_json` (uses orjson under the hood when configured); avoid `json.dumps` on hot paths.

---

## 16. Don'ts (review will reject)

- `print(...)` in production code.
- `# type: ignore` without a follow-up comment explaining why.
- `# TODO` without a linked issue number.
- `eval`, `exec`, `pickle.loads` on untrusted data.
- Catching `BaseException` (kills `KeyboardInterrupt`).
- `os.system(...)` or `shell=True`.
- Mutable default arguments (`def f(x: list = []):`).
- Importing `langchain` outside `llm/`.
- Importing `anthropic`/`openai` outside `llm/providers/`.
- Importing PyGithub outside `tools/github_tool.py`.

---

## 17. Recommended reading

- The [Pydantic v2 docs](https://docs.pydantic.dev/latest/) — every contributor uses these daily.
- The [structlog docs](https://www.structlog.org/) — particularly the contextvars chapter.
- The [LangGraph docs](https://langchain-ai.github.io/langgraph/) — focus on state reducers and subgraphs.
- [PEP 8](https://peps.python.org/pep-0008/) — for the rare cases ruff doesn't cover.
