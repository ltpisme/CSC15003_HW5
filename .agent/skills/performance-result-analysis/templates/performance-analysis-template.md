# Performance Analysis Output Template

```markdown
# Performance Analysis: ${SCENARIO_NAME}

## 1. Executive Summary Table

| Metric Field | Measured Value | Unit / Status |
| :--- | :--- | :--- |
| **Scenario** | ${SCENARIO_NAME} | Type |
| **Objective** | ${SCENARIO_OBJECTIVE} | Goal |
| **Configuration** | ${LOAD_CONFIGURATION} | VUs / Ramp / Duration |
| **Dataset** | ${DATASET_INFO} | CSV / Parameterization |
| **Total Requests** | ${TOTAL_SAMPLES} | Samples |
| **Success** | ${SUCCESS_COUNT} | Samples |
| **Errors** | ${ERROR_COUNT} | Samples |
| **Error Rate** | ${ERROR_RATE_PCT}% | Percentage |
| **Throughput** | ${THROUGHPUT_RPS} | req/s |
| **Average** | ${MEAN_RES_TIME_MS} | ms |
| **Median** | ${MEDIAN_P50_MS} | ms |
| **P90** | ${P90_PCT1_MS} | ms |
| **P95** | ${P95_PCT2_MS} | ms |
| **P99** | ${P99_PCT3_MS} | ms |
| **Min** | ${MIN_RES_TIME_MS} | ms |
| **Max** | ${MAX_RES_TIME_MS} | ms |
| **Resource Usage** | ${RESOURCE_OBSERVATION} | CPU / RAM / Disk |

> *Note:* If any metric is not present in the raw data, it must be marked as `N/A — not available in provided result`.

---

## 2. Request-Level Detailed Breakdown

| Sampler Name | Method | Total Samples | Errors (%) | Mean (ms) | Median (ms) | P90 (ms) | P95 (ms) | P99 (ms) | Min (ms) | Max (ms) | Throughput (req/s) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| `${SAMPLER_1}` | GET/POST | ... | ...% | ... | ... | ... | ... | ... | ... | ... | ... |
| `${SAMPLER_2}` | GET/POST | ... | ...% | ... | ... | ... | ... | ... | ... | ... | ... |

---

## 3. Behavioral Analysis & Findings

### Observed Behavior
${OBSERVED_BEHAVIOR_DESCRIPTION}

### Performance Finding
${PERFORMANCE_FINDINGS_DESCRIPTION}

### Risk Assessment
${IDENTIFIED_RISKS_OR_BOTTLENECKS}

---

## 4. Conclusion & Evidence Reference

* **Conclusion:** ${OBJECTIVE_CONCLUSION}
* **Evidence:**
  - Raw JTL: `${RAW_JTL_PATH}`
  - Aggregate Statistics: `${STATISTICS_JSON_PATH}`
  - Resource Screenshot: `${EVIDENCE_SCREENSHOT_PATH}`
```
