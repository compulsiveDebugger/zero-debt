# ADR-0004: Single `LLMGateway` Abstraction with Logical Model Ids

**Status:** Accepted

## Context

Five phases call LLMs for distinct tasks (fix proposal, test generation, code review, etc.). Each task has different needs — fast + cheap vs. deeper reasoning. If we sprinkle direct SDK calls across nodes:

- Swapping Claude ↔ OpenAI ↔ local models requires code changes in every node.
- Cross-cutting concerns (retry, rate limiting, token accounting, prompt caching, redacted logging) get re-implemented or skipped per call site.
- Testing requires mocking SDKs at multiple points.

Model selection decisions also shouldn't be hardcoded — a security-review prompt might want the deepest model available; a ranking call might want the fastest.

## Decision

All LLM calls route through a single `LLMGateway` abstract class. Nodes request models by **logical id** (`fast`, `code-smart`, `review-deep`), not vendor name. The gateway config maps logical id → concrete provider + model.

- `LLMGateway.complete(LLMRequest) -> LLMResponse` is the only method nodes call.
- Provider-specific implementations (`AnthropicGateway`, `OpenAIGateway`, etc.) live under `llm/providers/`.
- Capabilities (`json_mode`, `tool_use`, `vision`, `streaming`, `prompt_cache`) are probed via `gateway.supports("...")` so callers can gracefully degrade.
- Structured output enforcement is a first-class request field (`json_schema`), not a per-provider hack.
- Token usage, latency, and provider metadata are returned in `LLMResponse` — no hidden state, no callback hooks.

## Consequences

**Positive**

- Provider swap is a YAML change, zero code churn.
- Cross-cutting concerns land in one place: gateway-level retry (via `tenacity`), redacted logging, token accounting, cache handling.
- Tests mock a single abstraction instead of multiple SDKs.
- Phase authors think about *role* (what level of reasoning this call needs), not *vendor*.

**Negative**

- Thin layer of indirection between nodes and SDKs — one more file to read when debugging a call.
- Gateway is now a choke point; its reliability is the system's reliability. Covered by a dedicated test suite with `respx` replay fixtures.

**Discipline required**

- Node code **never** imports `anthropic` or `openai` directly. A lint rule / grep check in CI should flag these imports outside `llm/providers/`.
