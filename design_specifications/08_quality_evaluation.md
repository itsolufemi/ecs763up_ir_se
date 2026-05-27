# 08a. Quality and Evaluation (Simplified)

## 8.1 Offline evaluation

### Test collections
Instead of generic datasets (TREC), we build a **Domain-Specific Golden Set**:
-   **Size:** ~50 representative queries defined by the Art Curator.
-   **Types:**
    -   *Known-Item:* "The Starry Night" (Must return ID: 105).
    -   *Broad Topic:* "Impressionist landscapes" (Must return IDs: 101, 102, 105...).
    -   *Negative Test:* "Modernist cars" (Should return 0 results).

### Metrics
We focus on metrics that reflect the curator's daily experience:
-   **Precision@10:** How many of the top 10 results are actually relevant?
-   **Recall:** Did the search find the specific painting the curator was looking for?
-   **Latency:** Total request time (Target: < 100ms).


### Error analysis
We perform manual failure analysis on the Golden Set:
-   **Zero Results:** Did we miss a synonym? (e.g., "cubism" vs "cubist").
-   **Ranking Errors:** Why is a sketch ranked higher than the final painting? (Adjust boosting weights).

---

## 8.2 Online evaluation

Since this is a single-user internal tool, standard "Big Data" online evaluation methods are not applicable.

### A/B testing
-   **Status:** **Not Applicable.**
-   **Reasoning:** A/B testing requires thousands of users to detect statistical significance. We cannot A/B test on one user.
-   **Alternative:** **"Staging Review."** The developer lets the curator try the new ranking algorithm on a test version before making it live.

### Interleaving
-   **Status:** **Removed.**
-   **Reasoning:** Requires high query volume to determine preference between two rankers.

### Click debiasing
-   **Status:** **Removed.**
-   **Reasoning:** We do not have click logs. The curator's feedback is explicit (verbal/written), not implicit (clicks).

---

## 8.3 Human judgments

The **Curator** is the sole source of truth.

-   **Method:** Expert Review.
-   **Process:**
    1.  Developer runs the Golden Set queries.
    2.  Curator marks results as "Relevant" (1) or "Not Relevant" (0).
    3.  We tune the Sparse/Vector weights until the Curator is satisfied.
-   **Simplification:** No need for "Inter-annotator agreement" (Cohen’s κ) since there is only one judge.

---

## 8.4 Continuous evaluation (CI)

We implement a simple **Regression Test Script** (`evaluate.py`) that runs before any code update.

-   **Thresholds:**
    -   **Known-Item Accuracy:** Must be 100% (Specific titles must always be found).
    -   **Latency:** Alert if average search time exceeds 200ms.
-   **Action:** If the script fails, the update is rejected.