# 02a. Ingestion and Document Processing (Simplified)

## 2.1 Data Sources

While the core collection is internal, we enrich this data with targeted external sources to provide deeper context (e.g., critical analysis, video essays).

### Primary Sources (Internal)
1.  **Art Collection Database (Metadata):**
    *   **Format:** CSV export or direct SQL connection (SQLite/PostgreSQL).
    *   **Fields:** `id`, `title`, `artist`, `year`, `medium`, `dimensions`, `location`.
2.  **Curatorial Descriptions (Text):**
    *   **Format:** Text files, Word docs, or Markdown documents linked by `artwork_id`.
    *   **Content:** Detailed essays, exhibition history, visual analysis.
3.  **Digital Images (Visuals):**
    *   **Format:** JPEG/PNG files in a local directory.
    *   **Usage:** Displayed in results (thumbnails).

### Secondary Sources (External Enrichment)
4.  **Gallery Websites (Web Scraping):**
    *   **Target:** Specific museum pages (e.g., Tate, MoMA) relevant to our collection.
    *   **Content:** Provenance, exhibition history, and critical reviews.
5.  **Multimedia (YouTube):**
    *   **Target:** Educational art channels (e.g., Smarthistory, Great Art Explained).
    *   **Content:** Video transcripts/captions used as searchable text.

---

## 2.2 Processing Workflow (Batch Script)

Instead of a complex streaming distributed system, we use a linear **Batch Processing Script** (e.g., a Python script `build_index.py`).

### Workflow Steps
1.  **Load:** Read the CSV catalog and load associated text descriptions into memory.
2.  **Pre-process:** Apply text normalization (lowercase, remove punctuation) and tokenization.
3.  **Enrich:** Pass descriptions through the BERT model to generate dense vector embeddings.
4.  **Index:**
    *   Build the **Sparse Index** (Inverted Index dictionary).
    *   Build the **Vector Index** (Save embeddings to a NumPy array).
5.  **Save:** Serialize the indices to disk (e.g., `index.json`, `vectors.npy`).

### Operational Properties
-   **Run Frequency:** Ad-hoc (whenever the collection is updated).
-   **Error Handling:** Log invalid records to `ingestion_errors.log` and continue.
-   **State:** Stateless run; rebuilds the index from scratch each time (feasible for <2,000 docs).

---

## 2.3 Extraction

Simple text extraction logic:
-   **Structured Data:** Read directly from CSV columns.
-   **Unstructured Text:** Read file content.
-   **Web Content:** HTML parsing (e.g., BeautifulSoup) to strip navigation/ads and retain article text.
-   **Video Content:** Download auto-generated captions (VTT/SRT) and merge into a single text block.
-   **Output:** A standardized dictionary object per artwork:
    ```json
    {
      "id": "101",
      "title": "Water Lilies",
      "description": "Oil on canvas... Impressionist style...",
      "metadata": {"artist": "Monet", "year": 1919}
    }
    ```

---

## 2.4 Normalization and Linguistic Analysis

Standard IR pipeline (aligned with Lecture 2):

1.  **Normalization:** Lowercase all text; remove accents (e.g., "Mondrian" matches "Mondrián").
2.  **Tokenization:** Split text on whitespace and punctuation.
3.  **Stop-word Removal:** Remove common English words (the, and, of) to reduce index size.
4.  **Stemming:** Use a standard algorithm (e.g., Porter Stemmer) to map "painting" and "painted" to "paint".

---

## 2.5 Enrichment (Metadata)

-   **Named Entity Recognition (NER):** (Optional) Identify Artist names and Historical Periods if not explicitly in metadata.
-   **Keyphrase Extraction:** Extract top TF-IDF terms for the "Quick Summary" view.

---

## 2.6 Chunking Strategy

Required for the Vector (BERT) index, as models have a token limit (usually 512 tokens).

-   **Method:** Split long curatorial descriptions into smaller passages (e.g., 2-3 sentences or 100 words).
-   **Overlap:** Include small overlap (e.g., 20 words) so context isn't lost at the cut.
-   **Storage:** Each chunk is embedded separately but links back to the parent `artwork_id`.

---

## 2.7 Embedding Generation

-   **Model:** Use a BERT Transformer model (e.g., `all-MiniLM-L6-v2`) via the `sentence-transformers` library.
-   **Output:** A 384 or 768-dimensional vector for every document/chunk.

---

## 2.8 Quality Gates

-   **Validation:** Ensure every artwork has at least a Title and an ID.
-   **Logging:** Warn if a description is empty or if an image file is missing.
