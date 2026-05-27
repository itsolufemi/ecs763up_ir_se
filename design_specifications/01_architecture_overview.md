# 01a. System Architecture Overview (Simplified for Art Gallery Use Case)

## 1.1 Goals and non-goals

### Goals
- **Relevance:** Accurately retrieve artworks by title, artist, or semantic description using a hybrid of keyword (BM25F) and semantic (BERT) search.

- **Low latency:** p95 < 200 ms for typical queries.
- **Scale:** Designed for the size of art gallery's internal collection which would average ~2,000 documents. This exact-search architecture remains viable up to approximately **100,000 to 500,000 documents** on standard hardware before query latency exceeds the 200ms budget, at which point switching to ANN would be required.
- **Freshness:** NRT indexing for updates to artwork metadata.
- **Extensibility:** Easy to add new ranking features or metadata fields. 
- **Operational excellence:** Observability, resilience to partial failures.

### Non-goals (explicitly out of scope for this design)
- Scaling to millions of documents (this would require switching to ANN).
- Real-time conversational agent behavior.

---

## 1.2 Layers and responsibilities

```
┌───────────────────────────────────────────────────────────────────────────┐
│                               CLIENTS                                     │
│  Curator Dashboard (Console UI)                                           │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │ HTTPS
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                          EDGE / API GATEWAY                               │
│  Internal Auth • Request validation                                       │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                           SEARCH FRONTEND                                 │
│  Query parsing • rewriting • retrieval orchestration • fusion             │
│  snippets/highlights • response shaping                                   │
└───────────────┬───────────────────────────────┬───────────────────────────┘
                │                               │
                │                               │
                ▼                               ▼
┌───────────────────────────────┐     ┌─────────────────────────────────────┐
│  SPARSE RETRIEVAL SERVICE     │     │  VECTOR RETRIEVAL SERVICE           │
│  (BM25F over inverted idx)    │     │  (Exact k-NN over embeddings)       │
│  (Single Node)                │     │  (Single Node)                      │
└───────────────┬───────────────┘     └─────────────────┬───────────────────┘
                │                                         │
                └───────────────┬─────────────────────────┘
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         RERANKING / LTR SERVICE                           │
│  Cross-encoder (optional)                                                 │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                              RESPONSE                                     │
│  Ranked docs + facets + snippets + “why this result”                      │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 1.3 Ingestion / indexing architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                    │
│  Art Gallery DB • Curated CSVs • Artist Biographies                       │
└───────────────┬───────────────────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         DATA LOADERS                                      │
│  Read CSV • Load Images • Parse Text                                      │
└───────────────┬───────────────────────────────────────────────────────────┘
                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                    DOCUMENT PROCESSING PIPELINE                           │
│  Extract → Normalize → Enrich → Chunk → Embed → Quality checks            │
│                                                                           │
└───────┬───────────────────────┬───────────────────────────┬───────────────┘
        │                       │                           │
        ▼                       ▼                           ▼
┌──────────────┐        ┌───────────────── ┐       ┌───────────────────────┐
│ Doc Store    │        │ Sparse Indexer   │       │ Vector Indexer        │
│ (raw+clean)  │        │(inverted/forward)│       │ (Flat Index/Numpy     │
│              │        │                  │       │    array)             │
└──────┬───────┘        └──────────┬───────┘       └──────────┬────────────┘
       │                           │                          │
       ▼                           ▼                          ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         INDEX STORAGE + SNAPSHOTS                         │
│  shard files • segment merges • replication • backups • versioning        │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 1.4 Request flow (online)

1.  **Edge:** Internal authentication (login check), validate query.
2.  **Query processing:** Parse and normalize text (lowercase, remove punctuation).
3.  **Candidate generation:** Run sparse (BM25F) and vector (Exact k-NN via BERT Transformers) retrieval.
4.  **Fusion:** Combine the two lists using a simple weighted sum or RRF.
5.  **Reranking (Optional):** Re-score top results (if needed).
6.  **Presentation:** Generate snippets and format the final JSON response.

---

## 1.5 Deployment topology (practical)

- **Frontend Service:** Single stateless service (e.g., Python/FastAPI).
- **Sparse Service:** Local index (e.g., using a library like Rank-BM25 or Lucene).
- **Vector Service:** In-memory flat index (NumPy or FAISS); runs on CPU.
- **Data Storage:** Local Database (SQLite/PostgreSQL) for metadata; local disk for document files.

---

## 1.6 Design Rationale for Small-Scale Vector Search

For the target collection size of ~2,000 documents, **Approximate Nearest Neighbor (ANN)** search is not required. We will use **Exact k-Nearest Neighbor (k-NN)** search instead.

- **Mechanism:** This involves a brute-force comparison of the query vector against all 2,000 document vectors using cosine similarity.
- **Justification:**
    - **Perfect Accuracy:** It guarantees finding the true nearest neighbors, unlike ANN which trades some accuracy for speed.
    - **Sufficient Speed:** A modern CPU can perform thousands of vector comparisons in milliseconds, well within our latency budget.
    - **Simplicity:** It avoids the complexity of building and tuning an ANN index (e.g., HNSW graphs), simplifying the implementation.

This approach is "right-sized" for the project's scale, prioritizing simplicity and accuracy where extreme scalability is not a primary goal.