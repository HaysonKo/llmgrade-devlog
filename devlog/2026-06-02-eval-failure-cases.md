# Eval design is a failure taxonomy, not a score

## Context

LLMGrade started as a simple regression check for a RAG support bot: score a baseline
answer and a candidate answer against the source policy, block the merge if the candidate
regressed. One case, one number.

Then the demo became editable — you can paste your own source, question, and two outputs.
That immediately exposed the real problem: assigning a score is the easy part. The hard
part is deciding *what kind of answer this is* before you score it.

## Test

We ran a spread of candidate outputs against the refund-policy case (policy: returns within
30 days, no exceptions listed):

- hallucinated policy exception — `Yes, you can return it after 45 days if you were sick. Medical exceptions are allowed.`
- safe refusal — `No, you cannot return it after 45 days if you were sick. No medical exceptions are listed in the policy.`
- bare refusal — `No.`
- copied source truth — the source policy pasted back as the answer
- invalid output — `d;lakjld;sad`

## Observation

A single scoring formula treats all of these as "an answer" and hands back a plausible
number. The invalid string scored about the same as a bare refusal. The safe refusal was
penalized for *mentioning* "medical exceptions" even though it denied them. Grounded-looking
text and meaningless text landed in the same band.

## Problem

These are different failure categories, and they need different outcomes — not just
different numbers:

- **hallucinated policy** should **critical-fail** (it invents an exception the policy never states)
- **safe refusal** should **pass** (correct, grounded, declines cleanly)
- **bare refusal** should **not critical-fail**, but should **score lower** (no grounding, no next step)
- **copied source truth** can be **grounded but incomplete** (it doesn't answer the question)
- **invalid output** should be **blocked as a non-answer** (nothing to evaluate — distinct from hallucination)

## Product decision

Add a deterministic preflight and a calibration taxonomy:

- detect invalid/non-answers before rubric scoring and collapse them to ~0, blocked, but
  **not** a critical hallucination
- distinguish negated/safe statements from affirmative invented claims
- treat a verbatim source paste as grounded-but-incomplete (high groundedness, low
  completeness) and let the gate decide
- pin every category to an expected score band with calibration cases in the smoke test

## Result

Current deterministic bands for the refund-policy demo:

| Category            | Band (/10) | Critical | Outcome              |
| ------------------- | ---------- | -------- | -------------------- |
| ideal answer        | 9.0–9.5    | no       | pass                 |
| safe refusal        | 8.0–8.5    | no       | pass                 |
| copied source truth | 7.0–8.2    | no       | grounded, incomplete |
| bare refusal        | 5.0–5.5    | no       | weak                 |
| hallucinated policy | 3.5–4.0    | **yes**  | blocked              |
| invalid / unrelated | 0.0–1.0    | no       | blocked (non-answer) |

## Remaining gaps

- Scoring is deterministic and tuned to the refund-policy demo. It is not a general semantic
  judge.
- Invalid detection is heuristic (symbol ratio + domain relation + yes/no cue); a fluent but
  off-topic sentence sharing a domain word could still slip through.
- Copied-source detection is token overlap, not meaning.
- **Planned (V1.1):** model-backed rubric judging for custom domains. Not built, no API calls today.
