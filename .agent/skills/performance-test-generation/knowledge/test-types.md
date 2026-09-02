# Performance Test Types & Workload Profiles

## 1. Load Testing
* **Goal:** Evaluate system behavior under normal, expected operational capacity.
* **Workload Characteristics:**
  - Moderate concurrency (e.g., 20–50 Virtual Users).
  - Smooth ramp-up phase (30–60s).
  - Extended steady-state duration (3–5 minutes).
  - Realistic pacing / think time ($\sim 500\text{ ms}$).
* **Success Criteria:** P95 response time within target SLO (e.g., $\le 200\text{ ms}$), Error rate $0.00\%$, CPU usage $\le 40\%$.

---

## 2. Stress Testing
* **Goal:** Determine the breaking point, maximum capacity limit, and degradation behavior of the system under extreme load.
* **Workload Characteristics:**
  - Multi-stage staircase progression ($25\% \to 50\% \to 75\% \to 100\%+$ of peak design limit).
  - Reduced think time (100–300 ms).
  - Continuous observation of throughput plateau and latency inflection.
* **Key Observations:** Point of throughput saturation, database write-lock contention, resource exhaustion limits.

---

## 3. Spike Testing
* **Goal:** Assess system resilience and recovery capability when traffic surges abruptly.
* **Workload Characteristics:**
  - Low baseline load (e.g., 5 VUs).
  - Sudden traffic surge ($10\times–20\times$ baseline) within 10 seconds.
  - Brief peak sustain (30–60s).
  - Fast ramp-down back to baseline and dedicated recovery observation window (60s).
* **Success Criteria:** Zero crashes, quick queue draining, latency immediately returning to baseline levels after surge ends.

---

## 4. Endurance / Soak Testing
* **Goal:** Detect memory leaks, socket exhaustion, database connection pool depletion, and performance degradation over long duration.
* **Workload Characteristics:**
  - Steady sustained load at nominal capacity (e.g., 20 VUs).
  - Long observation window (15 minutes for local benchmarks, 4–24 hours for staging).
  - Segmented time-window analysis (e.g., 3-minute or 15-minute comparison buckets).
* **Success Criteria:** Flat latency trend across all time windows, stable memory ceiling (RSS plateau), zero unhandled exceptions.
