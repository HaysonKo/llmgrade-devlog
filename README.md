# LLMGrade Devlog

**Public development notes for LLMGrade** — Catch LLM app drift before it hits production.

[→ Visit LLMGrade](https://llmgrade.vercel.app/)

---

## What is LLMGrade?

LLMGrade helps solo builders and small teams shipping RAG support bots **automatically detect regressions** when changing prompts, models, or retrieval.

It generates app-specific evaluation cases from your docs + prompt, scores changes with a rubric judge, and can block merges in CI if quality drops.


### Key Features

- App-specific eval generation (not generic benchmarks)
- Rubric-based grading (groundedness, policy accuracy, refusals, etc.)
- Diff mode (`llmgrade diff`) for PR checks
- Critical failure detection (e.g. invented policies)
- Works completely offline

## Quick Start

```bash
npx llmgrade diff --base main --head feature/new-prompt
