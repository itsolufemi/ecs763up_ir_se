# 04a. Query Processing (Simplified)

## 4.1 Inputs

The search interface (Curator Dashboard) sends a simple JSON payload:

```json
{
  "q": "oil paintings by monet 1919",
  "filters": {
    "artist": "Claude Monet",
    "year_start": 1900,
    "year_end": 1920
  }
}
```

---

## 4.2 Parsing and Normalization

We process the query in two ways to support our Hybrid Retrieval model.

### 4.2.1 For Sparse Search (BM25F)
To ensure the BM25F index finds matches, we apply the exact same normalization used during ingestion:
1.  **Lowercasing:** "Monet" $\rightarrow$ "monet".
2.  **Punctuation Removal:** Remove commas, dashes, etc.
3.  **Stop-word Removal:** Remove "by", "in", "the".
4.  **Stemming:** "paintings" $\rightarrow$ "paint".

**Output:** A list of tokens: `["oil", "paint", "monet", "1919"]`.

### 4.2.2 For Semantic Search (Vector)
We pass the **raw, natural language query** to the embedding model. We do *not* remove stop-words here, as BERT uses them to understand context.
1.  **Input:** "oil paintings by monet 1919"
2.  **Model:** `all-MiniLM-L6-v2` (Same as ingestion).
3.  **Output:** A 384-dimensional dense vector.

---

## 4.3 Spelling Correction

We do not need a complex AI model. We use a **Levenshtein Distance** check against our metadata dictionaries.

-   **Pipeline:**
    1.  Check if token exists in the Inverted Index.
    2.  If not (OOV), calculate edit distance against the `Artist` and `Medium` lists.
    3.  If distance $\le$ 1, suggest the correction.
-   **Example:** "Picaso" (unknown) $\rightarrow$ "Picasso" (known artist).

---

## 4.4 Autocomplete / Suggestions

For a small internal tool, we implement a simple **Prefix Match** strategy.

-   **Source:** A pre-computed list of unique Artist names and Titles.
-   **Mechanism:** As the user types, filter the list for strings starting with the input.
-   **UI:** Display top 5 matches to guide the curator to valid entity names.

---

## 4.5 Intent Classification (Rule-Based)

Instead of a machine learning classifier, we use **Heuristic Rules** to weight query terms (Term Weighting).

-   **Named Entity Recognition:** If a token matches a known Artist (e.g., "Monet"), we infer the intent is "Artist Search".
    -   *Action:* Boost the weight of that term by **2.0x** in the BM25F query.
-   **Technical Terms:** If a token matches a known Medium (e.g., "Oil"), boost by **1.5x**.
-   **General Search:** All other terms get standard weight **1.0x**.

---

## 4.6 Query Rewriting and Expansion

We use a simple **Synonym Dictionary** relevant to art history to expand the query.

-   **Mechanism:** Look up tokens in a predefined JSON file and append synonyms to the sparse query.
-   **Examples:**
    -   "cubist" $\rightarrow$ add "cubism"
    -   "imp" $\rightarrow$ add "impressionism"
    -   "b&w" $\rightarrow$ add "black and white", "monochrome"

---

## 4.7 Filter Handling

Since we have a small dataset (~2,000 docs), we apply filters **Post-Retrieval**.

1.  **Retrieve:** Get top 100 results from Sparse and Vector search.
2.  **Filter:** Iterate through the merged list and discard items that don't match the metadata filters (e.g., `year < 1900`).
3.  **Sort/Cut:** Return the top 10 remaining results.

This "Post-Filter" approach is computationally cheap at this scale and significantly simplifies the indexing architecture.