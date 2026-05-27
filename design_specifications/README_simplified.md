# Art Gallery Search Engine — Simplified Design (Hybrid Sparse + Dense)
**Date:** 2026-02-24

This package contains the design for a **local, single-node search engine** tailored for an Art Gallery Curator internal tool. It supports:
- **Hybrid Retrieval:** BM25F (Keyword) + Exact k-NN (Semantic/BERT).
- **Rule-Based Reranking:** Transparent boosting for titles and artists.
- **Simple Stack:** In-memory indices, local file storage, and Python-based batch processing.
- **Focus:** High relevance for ~2,000 documents without distributed complexity.

## Contents
The design is broken down into the following simplified modules:

- `00_executive_summary.md` — High-level overview of the design philosophy and 6-week plan.
- `01_architecture_overview.md` — Single-node architecture, request flow, and deployment.
- `02_ingestion_and_processing.md` — Batch ingestion script, text normalization, and BERT embedding.
- `03_indexing.md` — In-memory Inverted Index and NumPy-based Vector Index.
- `04_query_processing.md` — Query parsing, synonym expansion, and spell checking.
- `05_ranking_and_fusion.md` — Weighted sum fusion and rule-based boosting logic.
- `06_serving_and_api.md` — FastAPI design, latency budgets, and error handling.
- `07_storage_and_data_models.md` — Local file schemas (JSON/Numpy) and directory structure.
- `08_quality_evaluation.md` — "Golden Set" evaluation metrics (Precision@10) and expert review.
- `09_operations_security.md` — Local logging, exception handling, and basic security.
- `10_workplan_and_risks.md` — 6-week implementation schedule and risk mitigation.
- `diagrams/` — Simplified ASCII diagrams for Online and Ingestion flows.

## How to use
Start with `00_executive_summary.md` for the big picture, then proceed through the modules in order.

This design is intended to be implemented using standard Python libraries:
- **Sparse:** `rank_bm25` or `whoosh`
- **Dense:** `sentence-transformers` + `numpy`
- **API:** `fastapi` + `uvicorn`