# 07a. Storage and Data Models (Simplified)

## 7.1 Core stores

For a single-node deployment with ~2,000 documents, we replace distributed storage systems (like S3, Redis, or Elasticsearch) with the **Local Filesystem** and **In-Memory Structures**.

1.  **Source Data (Raw):**
    -   **Location:** `./data/raw/`
    -   **Content:** The master CSV catalog (`catalog.csv`) and a folder of images (`./images/`).

2.  **Index Storage (Serialized):**
    -   **Sparse Index:** `sparse_index.json` (Dictionary mapping terms to doc IDs).
    -   **Dense Index:** `vectors.npy` (NumPy binary file storing the embedding matrix).
    -   **Metadata:** `documents.json` (List of all document objects for lookup and snippet generation).

3.  **Logs:**
    -   **Location:** `./logs/server.log`
    -   **Content:** Simple text logs for errors and query latency.

4.  **Feature Store:**
    -   **Status:** **Removed.**
    -   **Reasoning:** We use rule-based reranking (Section 5.5), which calculates features on-the-fly (e.g., "Title Match"). We do not need a persistent store for historical ML features.

---

## 7.2 Versioning

We do not need complex "manifest pointers" or "rolling updates" for an internal tool.

-   **Build Strategy:** The `build_index.py` script generates new index files.
-   **Backup:** Before overwriting, the script renames the old files to `index_backup_{date}.json`.
-   **Reload:** The search service (API) is simply restarted to load the new files into RAM.

---

## 7.3 Schemas

We define simple JSON schemas for our data objects.

### Document Schema (Internal)
```json
{
  "id": "str",
  "title": "str",
  "artist": "str",
  "description": "str",
  "year": "int",
  "medium": "str",
  "image_path": "str"
}
```

### Search Request Schema (API)
```json
{
  "q": "str",
  "filters": {
    "year_min": "int",
    "medium": "str"
  }
}
```

### Search Response Schema (API)
```json
{
  "results": [
    {
      "id": "str",
      "score": "float",
      "snippet": "str",
      "metadata": { "artist": "str", "year": "int" }
    }
  ],
  "total_hits": "int"
}
```