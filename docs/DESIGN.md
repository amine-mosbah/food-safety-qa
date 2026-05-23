# Food Safety Q&A Generator — Design & Operations

**Date:** May 22, 2026 (v0.3)

> **v0.3 note.** The pipeline now reads a single source-of-truth CSV (`data/food_recall_incidents.csv`, 7,546 records) and lets the user pick the cluster at runtime via either the `fsqa` Python CLI or an n8n form. The old train/valid/test split files and pre-sliced cluster CSVs have been deleted — see the changelog at the bottom for the v0.2 → v0.3 migration.

A reusable 4-stage pipeline for synthetic Q&A generation and LLM-as-judge evaluation on the SemEval-2025 food recall corpus.

---

## 1. Project context

The broader research goal is a **food safety question-answering system** trained on synthetic Q&A pairs generated from the SemEval-2025 Task 9 food recall corpus.

The full research arc is roughly:

```
[Cluster corpus]
   ↓
[Generate synthetic Q&A]            ← this repo implements this stage
   ↓
[Expert validation]
   ↓
[Build retrieval index (optional RAG)]
   ↓
[Fine-tune a small LM on validated Q&A]
   ↓
[Evaluate downstream QA quality with the LLM-as-judge framework]
```

This repository implements **synthetic Q&A generation + objective evaluation** as a reusable, repeatable pipeline.

---

## 2. Dataset

**Source:** SemEval-2025 Task 9 — Food Hazard Detection (extended release)
**License:** CC BY-NC-SA 4.0
**Size:** **7,546 records**, period 1994–2022
**Official links:**
- GitHub: <https://github.com/food-hazard-detection-semeval-2025/food-hazard-detection-semeval-2025.github.io>
- Zenodo DOI: <https://zenodo.org/doi/10.5281/zenodo.10820657>
- Paper: <https://aclanthology.org/2025.semeval-1.325>
- CodaLab competition: <https://codalab.lisn.upsaclay.fr/competitions/19955>

**Note on the record count.** The SemEval paper reports 6,644 labeled records (the snapshot used in the competition). The release that ended up on Zenodo / GitHub was later extended to **7,546 records** with additional fields. The pipeline uses the extended release because it is (a) larger per cluster — especially helpful for long-tail categories like `migration` (14 records), (b) richer in metadata (`product-title`, `hazard-title`, difficulty scores, `language`, `semeval-split`), and (c) the canonical link future researchers will land on.

Each record has:
- `title` — short recall headline
- `text` — full recall announcement (the main source for Q&A generation)
- `country`, `year`, `month`, `day` — provenance
- `hazard`, `hazard-category` — what triggered the recall (128 hazards → 10 categories)
- `product`, `product-category` — what was recalled (1,142 products → 22 categories)
- `semeval-split` — preserves the original `train` / `valid` / `test` assignment if you ever need to reconstruct the splits

There is now a **single file** in `data/`:
- `data/food_recall_incidents.csv` — the official 7,546-row CSV
- `data/README.md` — schema, distributions, and the curl command to refresh from upstream

Clusters and samples are produced **at runtime** by the CLI (`fsqa sample`) or the n8n workflow form, not stored as separate files.

---

## 3. Clustering choice

**Default clustering:** `hazard-category`. The 10 expert-assigned categories are:

| Cluster                          | Train count | Notes                              |
|----------------------------------|-------------|------------------------------------|
| **allergens** (seed)             | 1,854       | Largest, structurally consistent   |
| biological                       | 1,741       | Listeria, Salmonella, E.coli       |
| foreign bodies                   | 561         | Plastic, metal, glass              |
| fraud                            | 371         | Mislabelling, adulteration         |
| chemical                         | 287         | Pesticides, heavy metals           |
| other hazard                     | 134         | Catch-all                          |
| packaging defect                 | 54          | Failed seals, leaks                |
| organoleptic aspects             | 53          | Taste/smell issues                 |
| food additives and flavourings   | 24          | Unauthorised additives             |
| migration                        | 3           | Chemical leach from packaging      |

**Rejected alternatives:**
- k-means / HDBSCAN on text embeddings — adds compute + hyperparameters, harder to reproduce.
- `product-category` (22 buckets) — finer-grained but mixes very different hazards.

**Seed cluster: `allergens`** (2,527 records in the v0.3 dataset). Chosen because the recall texts follow a near-template shape (*"Product X is being recalled due to undeclared allergen Y"*), which gives the LLM-as-judge a clean, predictable target.

**v0.3 cluster counts** (run `./fsqa explore` to see live numbers):

| Cluster | Records |
|---|---|
| biological | 2,557 |
| allergens (seed) | 2,527 |
| foreign bodies | 943 |
| chemical | 578 |
| fraud | 527 |
| other hazard | 187 |
| packaging defect | 100 |
| organoleptic aspects | 81 |
| food additives and flavourings | 32 |
| migration | 14 |

---

## 4. Pipeline architecture

Four sequential LLM calls per record. Each stage has its own system prompt (`prompts/0[1-4]_*.md`) and a `.md` rationale block explaining the design choices.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Stage 1 · Question Generation                            temp = 0.7      │
│  Input:  recall record (title + text + metadata)                          │
│  Output: { question, justification }                                      │
│  Key idea: the SYSTEM prompt encodes a 7-point "what makes a question    │
│            realistic" rubric. The model must justify each question per   │
│            this rubric ("explain why realistic" requirement).              │
└──────────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────────┐
│ Stage 2 · Required + Optional Items Extraction           temp = 0.0      │
│  Input:  recall record + question                                         │
│  Output: { required_items: [...], optional_items: [...] }                 │
│  Key idea: decompose the expected answer into atomic facts BEFORE the    │
│            answer is generated. Provides the objective rubric the judge  │
│            will score against ("required vs optional" requirement).        │
└──────────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────────┐
│ Stage 3 · Answer Generation                              temp = 0.0      │
│  Input:  recall record + question                                         │
│  Output: { answer }                                                       │
│  Key idea: COMPLETELY DECOUPLED from Stage 1 — different temperature,    │
│            no shared context window beyond record+question. The answer   │
│            model does NOT see the required/optional items, so its A is   │
│            not biased toward the rubric ("Q independent from A").         │
│  Escape hatch: returns "INSUFFICIENT_CONTEXT" if the recall does not     │
│            contain enough info — these get scored 0.0 by the judge and   │
│            flagged for manual review (often indicates a Stage 1 Q         │
│            that overshot the source).                                    │
└──────────────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────────────┐
│ Stage 4 · LLM-as-Judge                                   temp = 0.0      │
│  Input:  question + required_items + optional_items + candidate_answer    │
│  Output: { required_checks, optional_checks,                              │
│            required_coverage, optional_coverage,                          │
│            overall_score = 0.8·required + 0.2·optional,                  │
│            verdict (1 sentence) }                                         │
│  Key idea: semantic match per item (paraphrases count). Two-axis score   │
│            supports filtering by required-coverage = 1.0 vs optional.    │
│            ("LLM-as-judge to evaluate" requirement).                       │
└──────────────────────────────────────────────────────────────────────────┘
                              ↓
                    outputs/qa_dataset.xlsx
                    outputs/raw_runs.jsonl
```

---

## 5. Key design decisions (and rejected alternatives)

### 5.1 Why 4 stages, not 2 (Q+A together)

The original May 14 pipeline was a single LLM call that generated 4 Q&A pairs in one shot, with all metadata visible. This was rejected because:

- A single model generating Q and A in the same call **leaks the answer into the question** (e.g., the Q phrasing tracks the A phrasing).
- There was no objective way to score quality — only post-hoc heuristic filters (JSON parse, length range, INSUFFICIENT_CONTEXT).

The 4-stage pipeline:

- Decouples Q-gen from A-gen — different temperatures, different reasoning chain.
- Inserts the **required-items decomposition stage** in between, which gives the judge an objective reference rather than a vibe rating.
- Lets the judge stage score each pair on an interpretable 0.0–1.0 axis.

### 5.2 Why GPT-4o-mini

- Aligned with the FinNLP paper family for synthetic Q&A evaluation.
- $0.025 per record across all 4 stages → ~$1.25 for the seed pilot, ~$166 for full 6,644.
- Strong JSON-mode reliability and instruction following.
- If quality at scale needs an upgrade, the most cost-effective place to spend is **only Stage 4 (judge)**: swap to `gpt-4o` there for stricter scoring while keeping Stages 1–3 on `gpt-4o-mini`.

### 5.3 Why one question per record (vs N=4)

The earlier draft pipeline generated 4 questions per record covering 4 fixed types (FACTUAL_EXTRACTION, CLASSIFICATION_OR_RISK, CAUSAL_REASONING, MULTI_HOP). The May 15 brief simplified this:

- **One Q per record, one justification, one answer, one judge score.**
- 50 records → 50 rows in the Excel sheet, one row per Q&A pair.
- Diversity comes from the cluster spanning 10 sub-hazards, not from forcing 4 question types per record.

**Extension path:** To add fixed question types back, run the pipeline N times per record with different `question_type` injections into the Stage 1 system prompt — no code surgery needed beyond adding a CLI flag.

### 5.4 Why auto-extract required items (vs hand-curate)

Hand-curating required items for 50 records is feasible (~2 hours), but doesn't scale to the remaining 9 clusters and 6,594 records. Auto-extraction with a separate LLM call:

- Reuses the same model and infrastructure.
- Keeps the pipeline reproducible on any cluster.
- Is spot-checkable: the seed Excel sheet has the required/optional items in every row for audit.

### 5.5 Sampling strategy

Stratified across the top 10 sub-hazards inside the cluster: 4 records from each of the top 10 (40), plus 10 from the long tail. This:

- Avoids the sample being dominated by "milk and products thereof" (which is 588 / 1,854 = 32% of the cluster).
- Surfaces edge cases (sesame, mustard, sulphites) worth manual scrutiny.

Use `./fsqa sample` to re-sample any cluster.

---

## 6. How to use this repo

There are two equivalent ways to run the pipeline. Pick whichever matches your style.

### A. Python CLI (`fsqa`) — recommended

```bash
# 1. Install deps (typer + rich + openpyxl)
pip install -r requirements.txt

# 2. One-time: store the OpenAI key locally (~/.config/fsqa/openai-key, chmod 600)
./fsqa setup

# 3. Explore the dataset
./fsqa explore                              # by hazard-category (default)
./fsqa explore --by product-category --top 15
./fsqa explore --by country --top 20

# 4. Sample a cluster (writes samples/<value>_<n>.csv with stable record_ids)
./fsqa sample --by hazard-category --value allergens --n 50
./fsqa sample --by hazard-category --value biological --n 50

# 5. Run the 4-stage pipeline
./fsqa run --input samples/allergens_50.csv --out outputs/allergens.xlsx

# Single interactive command that walks through everything
./fsqa wizard
```

### B. n8n form-driven workflow — for non-Python users

1. Import `n8n/food-safety-qa-pipeline.json` into your n8n instance.
2. Attach OpenAI + Google Sheets credentials.
3. Open the form URL n8n exposes when the workflow is active.
4. Upload `data/food_recall_incidents.csv`, pick a hazard category, enter sample size + Sheet ID + tab name, submit.
5. Watch the pipeline run; results land in your Google Sheet.

### Scale to a full cluster

```bash
# 1. Sample the entire cluster (use --n larger than the cluster size to get all rows)
./fsqa sample --by hazard-category --value allergens --n 99999 --out samples/allergens_full.csv

# 2. Run — 2,527 records ≈ ~6 hours, ~$60 at gpt-4o-mini pricing.
#    Use nohup / screen / tmux for long runs.
./fsqa run --input samples/allergens_full.csv --out outputs/allergens_full.xlsx
```

### Tune the prompts

Edit any of the `prompts/0[1-4]_*.md` files. The CLI and n8n workflow re-parse them on each run — no code change needed. Keep the `## System Prompt` and `## User Prompt Template` headers; the parser keys off them. After editing the `.md` files, regenerate the n8n JSON via the bake-prompts step in the build pipeline, or hand-edit the OpenAI nodes in n8n directly.

---

## 7. Suggested next steps

In rough priority order:

1. **Review the seed Excel sheet.** Audit:
   - Are the questions realistic? (Read `justification` column.)
   - Are the required items correct?
   - Are the answers grounded?
   - Is the judge being too lenient / too strict?
2. **Iterate on the prompts** (`prompts/0[1-4]_*.md`) based on review feedback. Re-run the seed pilot — it's only $1.25.
3. **Run the same pipeline on each of the other 9 clusters** with `./fsqa sample` and `./fsqa run`. Collect 9 more 50-record Excel sheets for manual review.
4. **Scale up to full clusters** once the prompt is locked in.
5. **Index the corpus for RAG.** The recall texts are short (~350 words), so a flat-index FAISS or chromadb setup with `text-embedding-3-small` is plenty. Then add a retrieval step before Stage 3 (A-gen) so the answerer can pull supporting recalls — this becomes a retrieval-augmented version of the pipeline.
6. **Fine-tune a small model** (e.g., `Qwen2.5-3B-Instruct`, `Phi-3-mini`) on the validated Q&A pairs. Evaluate using the **same** LLM-as-judge framework — that's the FinNLP paper's evaluation protocol applied to food safety.

---

## 8. Open questions / unresolved decisions

| Question | Current default | Notes |
|---|---|---|
| Which model for the judge at scale? | `gpt-4o-mini` | Upgrade to `gpt-4o` if manual review disagrees with judge scores >20% of the time. |
| Should `INSUFFICIENT_CONTEXT` rows be excluded from the dataset? | Kept, scored 0.0 | They flag Stage 1 Q-gen overreach — useful as a "to-review" bucket. |
| Should the judge see the recall source text? | No (only Q + items + A) | Keeps the judge's job clean: rubric-based, not source-fishing. Open to revisit. |
| Cluster definition revisit? | `hazard-category` (10) | Could switch to text-embedding k-means for ~50 finer clusters once more cluster-level Q&A exists. |
| RAG over what? | Not yet | If indexing, use the `text` field with `text-embedding-3-small`. |

---

## 9. Files inventory (v0.3)

| Path | Purpose |
|---|---|
| `README.md` | Quick overview + CLI / n8n quickstart |
| `fsqa` | Shell wrapper so users can run `./fsqa <command>` from the repo root |
| `docs/DESIGN.md` | This document |
| `docs/meeting-notes.md` | Raw notes from the May 15 meeting |
| `prompts/01_question_generation.md` | Stage 1 system + user prompt + rationale |
| `prompts/02_required_items_extraction.md` | Stage 2 system + user prompt + rationale |
| `prompts/03_answer_generation.md` | Stage 3 system + user prompt + rationale (v0.2 ECHO rule) |
| `prompts/04_llm_as_judge.md` | Stage 4 system + user prompt + rationale |
| `pipeline/fsqa.py` | All CLI commands (setup / explore / sample / run / wizard) |
| `n8n/food-safety-qa-pipeline.json` | Form-driven n8n workflow (16 nodes) |
| `data/food_recall_incidents.csv` | Official SemEval extended release (7,546 rows) |
| `data/README.md` | Dataset provenance + schema + cluster distribution |
| `samples/` | Generated by `fsqa sample` (git-ignored) |
| `outputs/` | Generated by `fsqa run` (git-ignored except `.gitkeep`) |
| `requirements.txt` | typer + rich + openpyxl |

---

## 10. v0.2 → v0.3 changelog

**Refactor — single source of truth + CLI + form-driven n8n.**

- ❌ **Removed** `data/incidents_train.csv`, `data/incidents_valid.csv`, `data/incidents_test.csv`, `data/allergens_seed_50.csv`, `data/allergens_full.csv`. All five are gone — clusters and samples are now computed at runtime.
- ❌ **Removed** `pipeline/run_pipeline.py` and `pipeline/sample_cluster.py`.
- ✅ **Added** `data/food_recall_incidents.csv` (7,546 records — the official extended release).
- ✅ **Added** `pipeline/fsqa.py` — single CLI with `setup`, `explore`, `sample`, `run`, `wizard` subcommands. Uses typer + rich for a clean UX (live progress bar, cost tracker, summary table).
- ✅ **Rewrote** `n8n/food-safety-qa-pipeline.json` — it now starts with a **Form Trigger** that asks for: (i) the CSV file, (ii) the hazard category, (iii) sample size, (iv) Google Sheet ID, (v) tab name. No more hard-coded data paths or sheet IDs inside the workflow.
- 🔄 **Prompts unchanged** from v0.2 (including the Stage 3 ECHO KEY ENTITIES rule that lifted req_coverage from 0.89 → 0.96 in the v0.2 pilot).
