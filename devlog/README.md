# LLMGrade devlog

LLMGrade is an independent prototype for exploring product-specific LLM evaluation. The current
public demo uses deterministic scoring so the evaluation contract is inspectable: source facts,
rubric axes, baseline behavior, candidate behavior, hard failure rules, and merge-blocking logic.

The proper next step is model-backed rubric judging, human review, or both, especially for
subjective qualities like helpfulness, tone, reasoning quality, and policy nuance. This version
starts deterministic to understand the product workflow before adding another model into the
evaluation loop.

---

LLMGrade explores **app-specific evals for LLM applications** — catching product-behavior
regressions before merge.

This devlog documents the work behind the public demo:

- product decisions and why they were made
- how the deterministic eval behaves
- rubric changes and calibration
- failure cases we found and how they're handled
- remaining gaps and what's still unsolved

## Scope

- The **main product code is not included here.** This repo is documentation only.
- Examples are **simplified and scoped to the public demo** (the refund-policy RAG
  support-bot case). They are illustrative, not production data.

See `STYLE.md` for how posts are written.
