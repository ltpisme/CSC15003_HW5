# Apache JMeter Design Patterns & Best Practices

## 1. Thread Group Topologies

### A. Standard Load & Endurance Topology (Single Thread Group)
* **Components:** Single `ThreadGroup`.
* **Configuration:**
  - `num_threads`: Number of target concurrent virtual users (e.g., 20–50).
  - `ramp_time`: Gradual ramp-up duration (e.g., 30s) to avoid cold-start storm.
  - `duration`: Sustained steady-state duration (e.g., 180s for Load, 900s for Endurance).

### B. Staircase Stress Topology (Multi-Thread Group Pattern)
* **Components:** Multiple parallel `ThreadGroup` elements with staggered startup delays.
* **Pattern:**
  - Stage 1: 25 VUs, delay 0s, duration 300s.
  - Stage 2: 25 VUs, delay 50s, duration 250s (Cumulative: 50 VUs).
  - Stage 3: 25 VUs, delay 100s, duration 200s (Cumulative: 75 VUs).
  - Stage 4: 25 VUs, delay 150s, duration 150s (Cumulative: 100 VUs).
* **Advantage:** Native JMeter support without requiring third-party plugins.

### C. Spike & Recovery Topology (Dual Thread Group Pattern)
* **Components:** 2 parallel `ThreadGroup` elements:
  - `Baseline_Group`: 5 VUs continuous across total duration (170s).
  - `Surge_Group`: 75 VUs with startup delay 30s, rapid ramp 10s (total 80 VUs), hold 60s, rapid ramp-down 10s.

---

## 2. Essential Component Best Practices

### A. HTTP Request Defaults & Headers
* Define global target host and port via CLI properties `${__P(host,localhost)}` and `${__P(port,8080)}`.
* Attach `HeaderManager` at Thread Group level with default `Content-Type: application/json`.

### B. Timers & Think Time (Pacing)
* **Gaussian Random Timer:** $\mu = 500\text{ ms}, \sigma = 200\text{ ms}$ (Realistic human thinking between requests).
* **Uniform Random Timer:** Min $100\text{ ms}$, Max $300\text{ ms}$ (High-intensity automated stress).

### C. Extractors & Assertions
* **JSONPostProcessor:** Extract tokens and IDs directly via JSONPath (`$.token`, `$.id`).
* **ResponseAssertion:** Validate HTTP Status (`200`, `201`) and expected string patterns.

### D. XML Schema Integrity Rules
* **Crucial:** Always ensure `<assertionsResultsToSave>0</assertionsResultsToSave>` is integer `0` rather than `"false"` to prevent parser `NumberFormatException`.
* Use relative paths for CSV files (`data/user_profiles.csv`).
