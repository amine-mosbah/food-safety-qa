# Progress Meeting Notes — May 15, 2026

**Attendees:**
- Havva (assistant to Prof. Ali Dehghantanha)
- Dr. Fattane (her supervisor)
- The next research intern (taking over this project)
- Amine Mosbah (current intern)

**Context:** Amine is moving to his primary Mitacs project (GraphGuard: Visual Analytics for AI-Security Telemetry). This meeting transferred the Food Safety Q&A project to a new intern and reset the pipeline design under Fattane's direction.

---

## Direction from Dr. Fattane

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
- If we **index** the corpus, we can use **RAG** (or MuRAG for multimodal — not needed here, text only).

### Build the full pipeline
Not just generation. Full pipeline = clustering → seed → Q-gen → required items → A-gen → judge.

---

## Deliverables for handover

1. **API key** working and configured.
2. **Pipeline** code, prompts, sample outputs, documentation.
3. **GitHub repo** with everything the next intern needs.

---

## Decisions taken after the meeting (Amine + Clawdius)

| Decision           | Value                                           | Reasoning                                                                                          |
|--------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Cluster definition | `hazard-category` column (10 expert-labelled clusters) | Zero compute, expert-labelled, unambiguous → cleanest forward path for handover                  |
| Seed cluster       | `allergens` (1,854 records, 36.5% of train)     | Largest, structurally consistent ("undeclared X allergen"), easiest required-items to evaluate     |
| Sample size        | 50 records                                      | Per Fattane                                                                                        |
| Sampling strategy  | Stratified across top 10 sub-hazards (4 each + leftover) | Diversity inside the cluster, avoids over-representing "milk and products thereof"           |
| Model              | OpenAI `gpt-4o-mini`                            | TA recommended GPT (FinNLP-paper alignment); 4o-mini handles all 4 stages well, dirt cheap         |
| Output format      | xlsx (openpyxl)                                 | Per Fattane                                                                                        |
| Required-items     | Auto-extracted by LLM (Stage 2)                 | Scales to other clusters without hand-curation; spot-checked on seed                               |
| Justification      | One sentence per question, co-generated         | Per Amine's preference                                                                             |
