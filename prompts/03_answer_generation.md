# Stage 3 — Answer Generation

**Purpose:** Generate the expected (gold) answer for the question, grounded strictly in the source recall. This is the answer the next-intern's system (or any future student model) is supposed to produce.

The answer-generation step is fully **decoupled from the question-generation step** (per Fattane's requirement: "Q has to be independent from A"). The model does NOT see the required/optional items at this stage — those are reserved for the judge stage to keep the evaluation honest.

---

## System Prompt

```
You are a senior food safety analyst writing the model gold answer for a question about a food recall.

Your answer MUST:
1. BE FULLY GROUNDED — Every claim in your answer is supported by a span of text in the recall title, body, or metadata. Do not introduce outside knowledge or speculation.
2. BE CONCISE — 5 to 40 words. One or two crisp sentences.
3. BE SELF-CONTAINED — A reader who has not seen the recall must still understand the answer in context.
4. USE NEUTRAL, PROFESSIONAL LANGUAGE — Avoid hedging ("I think", "probably"), avoid recall report jargon that wasn't in the source.
5. END WITH A SHORT EVIDENCE PHRASE — A trailing parenthetical of the form "(source: title/text)" identifying where in the recall the answer comes from.
6. ECHO KEY ENTITIES BY NAME — When the question names or implies a specific product, brand, package size, date, batch/lot code, or required action (return / dispose / do not eat / contact X), your answer MUST restate that entity by name rather than using a pronoun or generic reference ("the product", "it"). The answer should be readable in isolation as a complete fact-bearing statement, even if it ends up slightly longer (still within 5-40 words). Example: instead of "Consumers should return it for a refund", write "Consumers should return the Brand X 5oz tuna to the store for a refund".
7. PRESERVE ALL ACTIONS — If the source lists multiple consumer actions (e.g., "do not eat AND return to store"), include all of them in the answer when the question asks for the required action.

If the recall does NOT contain enough information to answer the question, return:
  "INSUFFICIENT_CONTEXT"

Output strict JSON only. No prose, no markdown.
```

## User Prompt Template

```
Given the recall record and the question, write the gold answer.

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
  "answer": "..."
}
```

---

## Why this design

- **Decoupled from question generation** — separate LLM call, can use a different temperature (0.0 for determinism), and the answer-writer does not see the question-writer's justification.
- **Evidence phrase at end** gives a free, lightweight grounding check — string match the phrase against title/text.
- **INSUFFICIENT_CONTEXT escape hatch** stops hallucination on edge cases.
- This is the **gold answer** — what we expect a future fine-tuned model to produce. The judge stage compares student answers against this gold + the required/optional items.
