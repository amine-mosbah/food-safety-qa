# Progress Meeting Notes — May 15, 2026

May 15, 2026 — pipeline requirements documented below.

---

## Pipeline requirements

### Clustering
- Cluster the dataset (record-level or category-level).
- Stick with **one cluster** and generate ~50 example questions on it.

### Output format
- Excel file with these columns:
  - record
  - category
  - **justification** — why this question is good
  - the question itself
  - (later: the answer, required items, judge scores)

### System prompt
- The system prompt **must explain why a question is realistic**, not just say "generate a question".

### Generation flow
- Q must be **independent from A**.
- Pipeline order: generate question → evaluate question → generate expected response.
- Generate the **items that NEED to be in the answer** (required items).
- Distinguish **required vs optional** items.
- Use an LLM-as-judge to evaluate candidate answers against those items.
- Use the LLM-as-judge again to ask: "How is our system performing?"

### Optional path
- Optional: index the corpus for **RAG** (or MuRAG for multimodal — not needed here, text only).

### Build the full pipeline
Not just generation. Full pipeline = clustering → seed → Q-gen → required items → A-gen → judge.

---

## Deliverables

1. **API key** working and configured.
2. **Pipeline** code, prompts, sample outputs, documentation.
3. **GitHub repo** with everything needed to run and extend the pipeline.

---

## Decisions taken after the meeting

| Decision           | Value                                           | Reasoning                                                                                          |
|--------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Cluster definition | `hazard-category` column (10 expert-labelled clusters) | Zero compute, expert-labelled, unambiguous → simplest reproducible default                         |
| Seed cluster       | `allergens` (1,854 records, 36.5% of train)     | Largest, structurally consistent ("undeclared X allergen"), easiest required-items to evaluate     |
| Sample size        | 50 records                                      | Meeting spec                                                                                       |
| Sampling strategy  | Stratified across top 10 sub-hazards (4 each + leftover) | Diversity inside the cluster, avoids over-representing "milk and products thereof"           |
| Model              | OpenAI `gpt-4o-mini`                            | FinNLP-paper alignment; 4o-mini handles all 4 stages well, low cost                                |
| Output format      | xlsx (openpyxl)                                 | Meeting spec                                                                                       |
| Required-items     | Auto-extracted by LLM (Stage 2)                 | Scales to other clusters without hand-curation; spot-checked on seed                               |
| Justification      | One sentence per question, co-generated         | Co-generated for quality review of each question                                                   |
