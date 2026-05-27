# 00a. Executive Summary (Simplified)

## Design Philosophy
This report details the design of a **Hybrid Retrieval System** tailored for a search engine to retrieve information from a collection of an art gallery's catalog. The system prioritizes **relevance** and **simplicity** over internet-scale scalability. It combines the precision of keyword search (BM25F) with the semantic understanding of vector search (BERT), fused via a transparent rule-based ranking logic.

## Key Architecture Decisions
1.  **Hybrid Retrieval:** We employ a dual-path strategy:
    *   **Sparse:** BM25F for exact title/artist matching.
    *   **Dense:** BERT embeddings for conceptual matching (e.g., "moody landscape").
2.  **Simplified Stack:**
    *   **Single-Node:** The system runs as a local Python service, avoiding distributed complexity.
    *   **Exact Search:** Given the collection size (~2,000 items), we use exact k-NN instead of approximate algorithms (ANN), guaranteeing 100% recall with negligible latency (<10ms).
3.  **Transparent Ranking:** Instead of opaque Machine Learning models, we use **Rule-Based Boosting** (e.g., `Title Match = 1.5x`), allowing the curator to easily understand and debug search results.

## Implementation Plan
The project is structured as a **6-week agile implementation**:
*   **Weeks 1-2:** Ingestion pipeline and sparse baseline.
*   **Weeks 3-4:** Vector integration and hybrid fusion.
*   **Weeks 5-6:** Evaluation against a curator-defined "Golden Set" and UI refinement.

## Conclusion
This design delivers a modern, AI-enhanced search experience while adhering to strict engineering constraints suitable for a single-developer academic project. It eliminates unnecessary operational overhead (sharding, MLOps) to focus entirely on **search quality** and **user utility**.