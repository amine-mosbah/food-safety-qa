# Dataset

## `food_recall_incidents.csv`

The single source of truth for this project. **All scripts and the n8n workflow take this file as input.**

| | |
|---|---|
| **Records** | 7,546 |
| **Period** | 1994 → 2022 |
| **Source** | SemEval-2025 Task 9 — Food Hazard Detection (extended release) |
| **License** | CC BY-NC-SA 4.0 |
| **Official links** | [GitHub](https://github.com/food-hazard-detection-semeval-2025/food-hazard-detection-semeval-2025.github.io) · [Zenodo DOI](https://zenodo.org/doi/10.5281/zenodo.10820657) · [Paper](https://aclanthology.org/2025.semeval-1.325) · [CodaLab competition](https://codalab.lisn.upsaclay.fr/competitions/19955) |

### Why this single file (and not the train/valid/test splits)

The original SemEval paper reports **6,644** labeled records. The release that ended up on Zenodo / CodaLab was later extended to **7,546 records** with additional fields (`product-title`, `hazard-title`, difficulty scores, language, `semeval-split` column). We use the extended release because:

1. More data per cluster (especially helpful for the long-tail hazard categories like `migration` = 14 records).
2. The `semeval-split` column is preserved, so you can reconstruct the original train/valid/test splits at any time without storing three separate files.
3. The richer columns (`hazard-title`, `product-title`, difficulty scores) are useful for future intern work on RAG / fine-tuning.

### Schema

| Column | Type | Notes |
|---|---|---|
| `Unnamed: 0` | int | Original row index from Zenodo CSV. Kept verbatim. |
| `year`, `month`, `day` | int | Recall date. |
| `title` | str | Recall report title. |
| `text` | str | Full recall body. **This is the main source field for Q&A generation.** |
| `product` | str | Product name (free text). |
| `product-category` | str | One of 22 product categories (labeled). |
| `product-title` | str | Canonical product title (curator-normalized). |
| `product-category-difficulty` | float | Classification difficulty score. |
| `product-difficulty` | float | Same, finer-grained. |
| `hazard` | str | Specific hazard (e.g., "Listeria monocytogenes", "milk"). 128 unique values. |
| `hazard-category` | str | One of 10 hazard categories. **Default clustering dimension.** |
| `hazard-title` | str | Canonical hazard title. |
| `hazard-category-difficulty` | float | Classification difficulty score. |
| `hazard-difficulty` | float | Same, finer-grained. |
| `language` | str | Source language (mostly `en`). |
| `country` | str | ISO 2-letter country code. |
| `semeval-split` | str | Original split assignment (`train` / `valid` / `test`). |

### Cluster distribution (`hazard-category`)

| Cluster | Records |
|---|---|
| biological | 2,557 |
| allergens | 2,527 |
| foreign bodies | 943 |
| chemical | 578 |
| fraud | 527 |
| other hazard | 187 |
| packaging defect | 100 |
| organoleptic aspects | 81 |
| food additives and flavourings | 32 |
| migration | 14 |

### Refreshing this file

If a newer release is published on Zenodo or GitHub, drop the new CSV in this folder under the same filename. Both the Python CLI and the n8n workflow are version-agnostic — they just expect the SemEval schema above.

```bash
# Re-download the latest official release
curl -L -o data/food_recall_incidents.csv \
  https://github.com/food-hazard-detection-semeval-2025/food-hazard-detection-semeval-2025.github.io/raw/main/data/food_recall_incidents.csv
```
