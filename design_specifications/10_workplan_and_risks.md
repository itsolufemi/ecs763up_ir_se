# 10. Workplan and Risks (Simplified)

## 10.1 Milestones (6 weeks example)

### Weeks 1–2: Core Search
-   Choose libraries (e.g., Elastisearch, SentenceTransformers).
-   Implement end-to-end ingestion from CSV/text files.
-   Build baseline sparse retrieval and simple UI.

### Weeks 3–4: Hybrid Retrieval and Ranking
-   Implement vector embedding generation and exact k-NN search.
-   Implement weighted sum fusion.
-   Add rule-based boosting (title/artist match).

### Weeks 5–6: Evaluation and Refinement
-   Build a "golden set" of test queries.
-   Implement offline evaluation script (precision/recall).
-   Gather feedback from the art curator and tune weights.

---

## 10.2 Risks and Mitigations

-   **Relevance Issues:**
    -   Risk: The search engine returns irrelevant results or misses key artworks.
    -   Mitigation:
        -   Create a representative "golden set" of queries.
        -   Thoroughly evaluate precision and recall on the golden set.
        -   Tune sparse/vector weights and boosting rules based on curator feedback.

-   **Performance Issues:**
    -   Risk: The search engine is slow or unresponsive.
    -   Mitigation:
        -   Profile the code to identify performance bottlenecks.
        -   Optimize data structures and algorithms for speed.
        -   Ensure the entire index fits in memory.

-   **Data Quality Issues:**
    -   Risk: The art catalog contains errors or inconsistencies.
    -   Mitigation:
        -   Implement data validation checks during ingestion.
        -   Log and report any data quality issues to the curator.

-   **Dependency Issues:**
    -   Risk: A required Python library is unavailable or incompatible.
    -   Mitigation:
        -   Use a virtual environment to manage dependencies.
        -   Pin specific versions of all libraries in `requirements.txt`.

-   **Code Complexity Issues:**
    -   Risk: The codebase becomes too complex and difficult to maintain.
    -   Mitigation:
        -   Follow clean coding practices (clear variable names, comments).
        -   Break down the code into small, modular functions.
        -   Regularly refactor the code to improve readability and maintainability.