# 05a. Ranking and Fusion (Simplified)

## 5.1 Candidate Generation

We retrieve candidates from two sources in parallel to capture both exact matches and conceptual similarity.

### 1. Sparse Retrieval (BM25F)
-   **Algorithm:** BM25F.
-   **Input:** Normalized query tokens.
-   **Output:** Top 100 documents with BM25F scores.
-   **Why:** BM25F is used because it allows the engine to treat multiple document fields (Title, Artist, Description) with different importance levels while capturing exact keyword matches.

### 2. Dense Retrieval (Vector)
-   **Algorithm:** Exact k-Nearest Neighbors (Cosine Similarity).
-   **Input:** Query embedding (BERT vector).
-   **Output:** Top 100 documents with Cosine Similarity scores (0.0 to 1.0).
-   **Why:** Captures semantic meaning (e.g., "melancholy landscape" matches "gloomy countryside").

---

## 5.2 Score Normalization

Sparse and dense scores live on different scales. We normalize them to make them comparable.

-   **Method:** Min-Max Normalization.
-   **Formula:** $S_{norm} = \frac{S - S_{min}}{S_{max} - S_{min}}$
-   **Application:** Applied per query to the top 100 results from the sparse stream (Vector scores are usually already 0-1 or -1 to 1).

---

## 5.3 Fusion Strategies

We use a **Weighted Linear Combination** to merge the two lists. This is intuitive and easy to tune.

-   **Formula:** $Score_{final} = \alpha \cdot S_{BM25F\_norm} + (1 - \alpha) \cdot S_{Vector}$
-   **Weights:**
    -   $\alpha = 0.5$ (Default): Equal weight to keywords and meaning.
    -   *Tuning:* Increase $\alpha$ if exact title matches are being missed.

---

## 5.4 Feature Computation (For Reranking)

Instead of complex ML features, we compute simple boolean or count-based features for the top results.

**Query-Doc Features:**
-   **Title Match:** Does the query string appear exactly in the artwork title? (True/False)
-   **Artist Match:** Does the query contain the artist's name? (True/False)
-   **Year Match:** Does the query contain a year that matches the artwork's creation year?

**Document Quality:**
-   **Has Image:** Does the record have a high-resolution image available? (True/False)

---

## 5.5 Reranking Logic (Rule-Based)

We replace the "Learning-to-Rank" (LTR) model with a transparent **Rule-Based Scoring** system.

**Logic:**
-   Start with the Fusion Score from 5.3.
-   **Apply Multipliers:**
    -   If `Title Match` is True: Score $\times 1.5$
    -   If `Artist Match` is True: Score $\times 1.2$
-   **Apply Penalties:**
    -   If `Has Image` is False: Score $\times 0.8$ (Demote records without visuals)

This acts as a "Lite" version of LTR, optimizing the list based on business rules rather than training data.

---

## 5.6 Reranking Stages (Pipeline)

1.  **Stage A:** Sparse Retrieval (Top 100) + Vector Retrieval (Top 100).
2.  **Stage B:** Score Normalization (Min-Max).
3.  **Stage C:** Fusion (Weighted Sum) $\rightarrow$ Top 50 Combined Candidates.
4.  **Stage D:** Feature Computation (Check Title/Artist matches).
5.  **Stage E:** Rule-Based Reranking (Apply multipliers).
6.  **Stage F:** Final Sort and Return Top 10.

---

## 5.7 Snippets, Highlights, and Explanations

### Snippets
-   **Logic:** Find the window of text (e.g., 30 words) in the description with the highest density of query terms.
-   **Highlighting:** Wrap query terms in `<b>` tags for display in the UI.

### "Why this result?"
We provide simple transparency to the curator:
-   "Matched Keywords: 'Oil', 'Canvas'"
-   "Semantic Similarity: 89%"
-   "Boosted by: Title Match"