# Performance Metrics Knowledge Base

## Core Performance Metrics Reference

### 1. Throughput (Requests Per Second / RPS)
* **Definition:** The number of requests processed by the server per unit of time (typically seconds).
* **Formula:** $\text{Throughput} = \frac{\text{Total Requests}}{\text{Total Elapsed Time (s)}}$
* **Interpretation:** Indicates the processing capacity of the System Under Test (SUT). Higher is better until resource saturation occurs.
* **Common Mistake:** Confusing client-side generated RPS with actual server-processed throughput during network or connection bottlenecks.

---

### 2. Response Time (Latency)
* **Definition:** Time elapsed from sending the first byte of a request until receiving the complete response body.
* **Components:** $\text{Response Time} = \text{DNS} + \text{TCP Handshake} + \text{TLS} + \text{Server Processing (TTFB)} + \text{Content Download}$.
* **Key Aggregates:**
  - **Mean (Average):** Arithmetic average. Easily distorted by outliers.
  - **Median (P50):** The 50th percentile. 50% of users experience response times $\le$ Median.
  - **P90 (90th Percentile):** 90% of requests are faster than this value.
  - **P95 (95th Percentile):** Standard industry metric for Service Level Agreements (SLAs).
  - **P99 (99th Percentile):** Measures the worst-case experience for 1% of transactions (Tail Latency).
  - **Min / Max:** The fastest and slowest recorded sample durations.
* **Common Mistake:** Relying solely on the Arithmetic Mean (Average) instead of P95/P99. An average of 100ms can hide significant 5-second spikes affecting 5% of users.

---

### 3. Error Rate
* **Definition:** The percentage of failed requests relative to total requests sent.
* **Formula:** $\text{Error Rate} = \frac{\text{Failed Requests}}{\text{Total Requests}} \times 100\%$
* **Failure Classification:**
  - HTTP 4xx (Client errors / Bad Request / Unauthorized / Rate Limited).
  - HTTP 5xx (Internal Server Error / Bad Gateway / Service Unavailable).
  - TCP / Network Socket Timeouts / Connection Refused.
  - Assertion Failures (Invalid JSON body / Missing response token).
* **Common Mistake:** Assuming 0.00% error rate implies acceptable performance when P95 latency has degraded to intolerable levels.

---

### 4. System Resource Metrics
* **CPU Utilization (%):** Total and per-core CPU load. $>80\%$ sustained indicates CPU saturation.
* **Memory (RSS / Heap MB):** Resident Set Size of the process. Monotonically increasing RSS under steady load indicates a potential Memory Leak.
* **Disk I/O & Lock Contention:** Disk write queue length and file lock serialization (especially prevalent in single-file databases like SQLite).
* **Network Bandwidth (KB/s):** Ingress (Received) and Egress (Sent) network throughput.
