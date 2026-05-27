# 03a. Indexing Design (Simplified)

## 3.1 Index Components

For a collection of ~2,000 documents, the entire index fits comfortably in memory (RAM). We maintain three simple data structures:

1.  **Sparse Index (Keyword):** A standard Inverted Index mapping terms to document IDs.
2.  **Dense Index (Semantic):** A simple 2D Matrix (NumPy array) storing document embeddings.
3.  **Metadata Store:** A structured table (DataFrame or SQLite) for filtering by Artist, Year, etc.

---

## 3.2 Sparse Index (Inverted)

### Structure
-   **Dictionary:** A Hash Map mapping `term` $\rightarrow$ `postings_list`.
-   **Postings List:** A simple list of `doc_id`s containing the term.
-   **Scoring Data:** To support BM25F, we store:
    -   `doc_len`: Length of each document.
    -   `avg_doc_len`: Average length across collection.
    -   `doc_freq`: How many documents contain a specific term.

### Simplification
-   **No Compression:** At this scale, raw integers are fine. We don't need delta-encoding or varints.
-   **No Skip Pointers:** Linear scan of postings lists is fast enough for <2,000 items.

---

## 3.3 Fielded Indexing (BM25F)

We maintain separate inverted indices (or separate dictionaries) for specific fields to allow for query-time boosting:

-   **`title`:** High boost (e.g., 2.0). Exact matches here are likely what the curator wants.
-   **`description`:** Standard weight (1.0). Used for semantic matching and general keyword search.
-   **`artist`:** Treated as a keyword field (exact match).

---

## 3.4 Dense Index (Flat / Exact)

### Structure
-   **Format:** A single dense matrix of shape `(N_documents, D_dimensions)`.
    -   $N \approx 2000$
    -   $D = 384$ (for `all-MiniLM-L6-v2`) or $768$ (for `BERT-base`).
-   **Storage:** Saved as a binary file (e.g., `.npy` or `.bin`).

### Search Algorithm
-   **Exact k-NN:** We do not use HNSW or IVF. We perform a brute-force dot product between the query vector and the entire document matrix.
-   **Why:** It guarantees 100% recall (perfect accuracy) and takes <5ms on a standard CPU.

---

## 3.5 Metadata Index (Filtering)

Used for "Faceted Search" (e.g., "Show me Oil paintings from 1900-1950").

-   **Implementation:** In-memory table (e.g., Python Dictionary or Pandas DataFrame).
-   **Fields:** `year` (int), `medium` (keyword), `location` (keyword).
-   **Operation:** Boolean filters applied *before* or *after* ranking.

---

## 3.6 Update Strategy (Batch)

-   **No Real-Time Segments:** We do not need complex "segment merging" or "tombstones."
-   **Full Rebuild:** When the catalog changes (e.g., once a week), we simply re-run the `build_index.py` script to regenerate the index files from scratch.

---

## 3.7 Capacity Planning

The hardware requirements for this design are trivial:

-   **Sparse Index:** ~2,000 docs $\times$ ~200 words/doc $\approx$ 400KB text. Index size < 5MB.
-   **Dense Index:** 2,000 docs $\times$ 768 dims $\times$ 4 bytes (float32) $\approx$ 6 MB.
-   **Total RAM:** < 50 MB. This can run on any basic laptop or free-tier cloud instance.
```