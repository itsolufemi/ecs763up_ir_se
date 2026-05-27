# 06a. Serving Layer and APIs (Simplified)

## 6.1 API Design

For this internal tool, we expose a single, straightforward REST endpoint via a lightweight framework like **FastAPI** or **Flask**.

### Search Endpoint
-   **URL:** `POST /search`
-   **Request Body:**
    ```json
    {
      "query": "impressionist sunrise",
      "filters": {"year_min": 1870, "medium": "oil"},
      "page": 1,
      "page_size": 10
    }
    ```
-   **Response:**
    ```json
    {
      "results": [
        {
          "id": "101",
          "title": "Impression, Sunrise",
          "snippet": "Famous <b>impressionist</b> work...",
          "score": 0.85,
          "explanation": "Title match (1.5x boost)"
        }
      ],
      "total_hits": 42
    }
    ```

---

## 6.2 Latency Budget

Given the dataset size (~2,000 documents) and in-memory architecture:
-   **Target:** < 100ms total response time.
-   **Breakdown:**
    -   Vector Search (Exact k-NN): < 10ms
    -   Sparse Search (BM25F): < 10ms
    -   Fusion & Reranking: < 5ms
    -   Network/Overhead: ~20-50ms
-   **Conclusion:** No complex optimization or caching layers are required to meet user expectations.

---

## 6.3 Caching Strategy

-   **Strategy:** **None.**
-   **Reasoning:** The entire index resides in RAM. Computing results from scratch for every query is computationally negligible for this scale. Adding a cache layer (like Redis) would add unnecessary architectural complexity.

---

## 6.4 Pagination

-   **Method:** Simple **Offset/Limit** pagination.
-   **Implementation:** `results[offset : offset + limit]`
-   **Why:** "Deep paging" performance issues do not exist for a result set maxing out at ~2,000 items. We do not need complex "search-after" cursors.

---

## 6.5 Security & Deployment

-   **Access:** Basic HTTP Authentication or network-level restriction (internal tool). No complex ACLs required.
-   **Runtime:** A single Python process (e.g., `uvicorn main:app`).
-   **State:** The application loads the `index.json` and `vectors.npy` files into global memory on startup.

---

## 6.6 Error Handling & Fallbacks

Even in a local deployment, components may fail (e.g., missing model files). We define a "Fail Soft" policy:

-   **Vector Search Failure:** If the embedding model fails to load or crashes, the system automatically falls back to **Sparse-Only (BM25F)** search.
-   **Reranker Timeout:** If the optional Cross-Encoder takes too long (>500ms), we skip the reranking step and return the initial fused list.
-   **Logging:** All application errors and fallbacks are logged to a local `server.log` file for debugging.