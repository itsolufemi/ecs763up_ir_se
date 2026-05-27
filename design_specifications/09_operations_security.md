# 09a. Operations, Reliability, and Security (Simplified)

## 9.1 Service Level Objectives (SLOs)

For a local, single-user tool, we define "Best Effort" objectives:

-   **Availability:** The tool should be available during working hours. If it crashes, a restart should take < 10 seconds.
-   **Latency:** Search results should appear in < 200ms to maintain a "snappy" feel.
-   **Freshness:** The index reflects the state of the CSV catalog at the last time `build_index.py` was run.

---

## 9.2 Observability

We rely on **Local Logging** rather than distributed metrics systems.

### Logging Strategy
-   **File:** All events are written to `app.log`.
-   **Format:** `[TIMESTAMP] [LEVEL] [COMPONENT] Message`
-   **Events to Log:**
    -   **Startup:** "Index loaded: 2045 documents."
    -   **Query:** "Search: 'Monet' | Hits: 12 | Time: 0.04s"
    -   **Errors:** Stack traces for any crashes (e.g., "Model file not found").

### Metrics
-   **Performance:** We do not need a dashboard (Grafana). We simply print the execution time of each search to the console for immediate feedback.

---

## 9.3 Reliability Patterns

Since there are no network calls between microservices, reliability is handled via **Exception Management**.

-   **Startup Checks:** The application verifies that `index.json` and `vectors.npy` exist before starting. If missing, it prompts the user to run the builder script.
-   **Graceful Degradation:**
    -   If the **Vector Model** fails to load (e.g., out of RAM), the system catches the exception, logs a warning, and starts in **"Keyword-Only Mode"** (BM25).
    -   This ensures the curator can still search by title even if the AI components fail.

---

## 9.4 Security

Security is delegated to the **Operating System** and **Network Isolation**.

-   **Network:** The API binds to `localhost` (127.0.0.1) only. It is not accessible from the outside internet.
-   **Access Control:** Access is restricted to the user logged into the physical machine.
-   **Data Privacy:** Since the art catalog is likely public information, no complex PII redaction is required.
-   **Input Validation:** Basic sanitization to prevent the search bar from crashing the server (e.g., limiting query length to 500 characters).

---

## 9.5 Incident Playbooks

Simple manual recovery steps for common issues:

1.  **"Search is slow":**
    -   *Action:* Restart the Python script to clear memory leaks.
2.  **"Results look wrong / Old data":**
    -   *Action:* Run `python build_index.py` to refresh the index from the latest CSV.
3.  **"Application won't start":**
    -   *Action:* Check `app.log`. If index files are corrupted, delete them and rebuild.