# Stage 2 — Required + Optional Answer Items Extraction

**Purpose:** Given a question + the source recall, decompose what a correct answer must contain. This gives the LLM-as-judge an objective rubric to score answers against (rather than vague "is this good?" prompting).

Required items = facts that MUST appear in the answer for it to be considered correct.
Optional items = facts that strengthen the answer but are not strictly necessary.

---

## System Prompt

```
You are a food safety evaluator. Your job is to look at a question about a food recall and decompose it into the minimal set of facts a correct answer MUST contain (required items), plus additional facts that strengthen the answer but are not strictly necessary (optional items).

Required items must satisfy ALL of these:
1. EXTRACTABLE — The fact is verifiable from the recall text or metadata provided.
2. ESSENTIAL — Omitting this fact from the answer would make the answer incorrect or seriously incomplete.
3. ATOMIC — Each item is a single fact (a contaminant name, a product name, a date, an action), not a sentence.

Optional items must satisfy:
1. EXTRACTABLE — Also verifiable from the source.
2. ENRICHING — Including these makes the answer more informative, but a competent answer without them is still acceptable.

Guidelines:
- Produce 1–3 required items per question. If a question is very narrow (e.g., "what hazard?"), 1 required item is fine. If it's a multi-hop question, 2–3 required items.
- Produce 0–5 optional items.
- Each item is a short string (1–8 words), not a sentence.
- Do NOT include items that are not present in the source recall.
- Do NOT include items that paraphrase the question itself.

Output strict JSON only. No prose, no markdown.
```

## User Prompt Template

```
Given the recall record and the question, extract the required and optional answer items.

###Recall record
- record_id: ${record_id}
- date: ${year}-${month}-${day}
- country: ${country}
- product: ${product}
- product-category: ${product_category}
- hazard: ${hazard}
- hazard-category: ${hazard_category}
- title: ${title}
- text: ${text}

###Question
${question}

###Output (strict JSON, no markdown):
{
  "required_items": ["...", "..."],
  "optional_items": ["...", "..."]
}
```

---

## Why this design

- **Decouples evaluation from generation** — by extracting the required-fact set independently of any candidate answer, the judge stage has an objective reference ("items that NEED to be in answer").
- **Atomic items** make string-matching feasible as a sanity check before LLM judging.
- **Required vs optional split** lets the judge compute two complementary metrics: required_coverage (must be 1.0) and optional_coverage (a soft score).
- This stage runs at temperature 0.0 — keeps extraction consistent and conservative, not creative.
