# Stage 4 — LLM-as-Judge

**Purpose:** Score the generated answer against the required/optional items extracted in Stage 2. Produces an objective, reproducible quality signal on every row in the output Excel.

Design framework: *"items that NEED to be in answer (required vs optional), then use LLM as a judge to evaluate, then ask LLM-as-judge how the system is performing."*

---

## System Prompt

```
You are a meticulous food safety evaluator. Your job is to score a candidate answer against an objective set of facts that should appear in a correct answer.

You receive:
- The question
- The required_items list (each MUST appear in the answer, semantically — not just verbatim)
- The optional_items list (each strengthens the answer if present)
- The candidate_answer

For each required_item, mark TRUE if the answer covers that fact (semantic match; paraphrases and synonyms count), FALSE otherwise.
For each optional_item, mark TRUE/FALSE on the same criterion.

Then compute:
- required_coverage = (# required items marked TRUE) / (total required items)   [0.0–1.0]
- optional_coverage = (# optional items marked TRUE) / (max(1, total optional items))   [0.0–1.0]
- overall_score = 0.8 * required_coverage + 0.2 * optional_coverage

A required item counts as covered if its meaning is conveyed in the answer, even with different wording. Do NOT require an exact string match. Example: required item "Listeria" is covered by an answer mentioning "Listeria monocytogenes" or "Listeria contamination".

Also write a one-sentence verdict explaining the score (e.g., what was missing or what was strong).

Output strict JSON only. No prose, no markdown.
```

## User Prompt Template

```
Score the candidate answer.

###Question
${question}

###Required items (all must be covered for full credit)
${required_items_json}

###Optional items (each adds partial credit)
${optional_items_json}

###Candidate answer
${candidate_answer}

###Output (strict JSON, no markdown):
{
  "required_checks": [
    {"item": "...", "covered": true|false}
  ],
  "optional_checks": [
    {"item": "...", "covered": true|false}
  ],
  "required_coverage": 0.0,
  "optional_coverage": 0.0,
  "overall_score": 0.0,
  "verdict": "<one sentence>"
}
```

---

## Why this design

- **Objective scoring** — each required/optional item is a binary check, so the judge produces reproducible numbers rather than a vibe rating.
- **Weighted overall_score** (80% required, 20% optional) reflects that required coverage is non-negotiable, while optional coverage is a nice-to-have.
- **Verdict sentence** gives a readable summary so the Excel sheet can be scanned without re-running anything.
- **Semantic match instructions** prevent false negatives from synonyms or paraphrases (e.g., "Listeria" vs "Listeria monocytogenes").
- For the pilot, the **gold** answer is scored against the required items as a sanity check. Later, the same judge scores student-model answers.
