# ADR-0005: Patch Validation is Structural, Never LLM Self-Eval

**Status:** Accepted

## Context

Phase 1 generates code patches via LLM. Some percentage of those patches will be wrong: hallucinated symbols, misapplied diff hunks, logically broken fixes, or fixes that introduce worse problems. We need a gate that prevents broken patches from being committed to a branch.

Two broad validation strategies exist:

1. **Semantic self-eval:** ask the same (or a different) LLM to grade the patch. Cheap to implement; deeply unreliable. Models are sycophantic about their own output and miss subtle failures, especially on code they just generated.
2. **Structural validation:** deterministic checks — does the diff apply? Does the result parse? Does lint regress? Does it compile?

## Decision

Every patch passes through a **structural validation pipeline** before `apply_patch` runs:

1. **Diff applicability** — `patch --dry-run` (or `unidiff`'s applier) must succeed cleanly.
2. **AST parse** — post-patch file must parse under tree-sitter (or language-native parser when available).
3. **Lint delta** — running the project's lint config on the patched file must not introduce new errors vs. pre-patch.
4. **Compile check (optional)** — when `config.validate_with_compile=true`, run the relevant incremental compile (`javac -Xstdout`, `tsc --noEmit`, etc.).
5. **Issue targeting** — the diff must overlap the Sonar-reported line ±3 lines (heuristic to catch patches that drift off-target).

No step in this pipeline asks an LLM "is this fix correct?" If a patch fails any gate, it's rejected with structured feedback that can be fed into the next retry's prompt.

## Consequences

**Positive**

- Determinism — given the same working tree and patch, validation result is identical across runs.
- No risk of the grader LLM and the generator LLM colluding in error.
- Fast — all gates are fully local; no extra API calls on the hot path.
- Clear failure modes — each gate has a distinct structured reason code, useful for retry prompts and audit reports.

**Negative**

- More infrastructure: tree-sitter grammars, per-language native toolchain invocation wrappers, lint config detection.
- Validation is structural only — it does not check whether the fix is *correct in intent*. A syntactically valid, lint-clean, compile-passing patch might still misunderstand the Sonar rule. Partial mitigation: the `addresses` field in `ProposedPatch` + overlap heuristic catches gross misses; fine-grained correctness relies on human PR review.

**Non-negotiable**

- We never flip to LLM-as-judge for this gate, even under pressure to "make the validator smarter." If we want semantic signal, we add a **separate** advisory score that's logged but not gating.
