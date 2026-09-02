# Result Analysis Methodology & Bottleneck Diagnostics

## 1. Multi-Tier Analysis Framework

### Tier 1: Ground Truth Ingestion (`raw.jtl`)
* Parse raw JTL log files line-by-line.
* Confirm that timestamps, elapsed milliseconds, response codes, and thread names match across all executions.

### Tier 2: Aggregate Cross-Check (`statistics.json`)
* Verify that JTL request totals and percentiles match the official dashboard `statistics.json`.
* If any discrepancy exists between AI summary text and `statistics.json`, the JSON/JTL values always supersede.

---

## 2. Bottleneck Classification & Diagnosis

### A. Database Write Contention & Lock Serialization
* **Symptom:** Disproportionately high P95/Max latency on write endpoints (`POST /register`, `POST /orders`) while read endpoints (`GET /products`) remain fast.
* **Root Cause:** Single-writer database locking (e.g., SQLite default Rollback Journal mode), unindexed table lookups, or unbatched transaction commits.

### B. In-Memory vs. Disk Performance Disparity
* **Symptom:** In-memory operations (e.g., RAM-stored cart) maintain sub-10ms response times under 100 VUs, whereas disk-persisted operations spike to multiple seconds.
* **Evaluation:** Confirms that application runtime logic is not CPU-bound; the bottleneck is disk I/O synchronization.

### C. Resource Saturation Patterns
* **CPU Saturation:** Sustained $>80\%$ CPU with rising response times across all endpoints.
* **Memory Leak:** Monotonically increasing RSS memory curve during sustained Endurance testing without reaching a plateau.
* **Connection Pool Exhaustion:** Abrupt spike in 504 Gateway Timeout or TCP connection errors when concurrency exceeds pool limits.
