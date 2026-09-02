---
name: performance-result-analysis
description: Automated analysis of JMeter performance test logs (*.jtl), aggregate statistics (*.json/HTML), and resource evidence to produce rigorous, evidence-backed performance audit reports and detect metric misinterpretations.
---

# Performance Result Analysis Skill

## Overview
This skill guides an AI agent through parsing raw JMeter execution logs (`*.jtl`), cross-verifying aggregate statistics (`statistics.json`), and evaluating visual resource monitoring evidence (`*.png`) to produce deterministic, hallucination-free performance reports.

---

## Input Requirements
* **Primary (Ground Truth):** Raw sample logs (`raw.jtl`).
* **Secondary (Aggregate Cross-Check):** JMeter HTML dashboard (`statistics.json` / `index.html`).
* **Supporting (Visual Verification):** Resource monitor screenshots (`evidence/*.png`), Hardware report.

---

## Strict Source of Truth Hierarchy

```
┌────────────────────────────────────────────────────────┐
│ 1. Raw JTL Logs (Request-level Ground Truth)           │
└───────────────────────────┬────────────────────────────┘
                            │ (Computes exact percentiles)
                            ▼
┌────────────────────────────────────────────────────────┐
│ 2. JMeter HTML / statistics.json (Aggregate Validation)│
└───────────────────────────┬────────────────────────────┘
                            │ (Cross-checks summaries)
                            ▼
┌────────────────────────────────────────────────────────┐
│ 3. PNG Screenshots (Visual & Hardware Corroboration)   │
└────────────────────────────────────────────────────────┘
```

> **CRITICAL RULE:** Never infer numeric performance metrics (Throughput, P95, Latency) from screenshot pixels when exact JTL / JSON data exists. If a metric is missing from the provided logs, explicitly record `N/A — not available in provided result`.

---

## Standard 5-Step Analysis Workflow

### Step 1: Ingest & Parse Raw Results
1. Load `raw.jtl` and `statistics.json` for all executed scenarios (Load, Stress, Spike, Endurance).
2. Extract total sample counts, successful samples, failed samples, error percentages, and elapsed execution window.

---

### Step 2: Calculate & Cross-Verify Metric Aggregates
Extract and verify request-level and total aggregates:
* **Throughput:** Total completed samples divided by total duration in seconds.
* **Latency Distribution:** Minimum, Mean (Average), Median (P50), 90th Percentile (P90), 95th Percentile (P95), 99th Percentile (P99), and Maximum Response Time.
* **Error Rate:** Percentage of HTTP 4xx/5xx or Assertion failures.
* **Bandwidth:** Received (KB/s) and Sent (KB/s).
* Consult `knowledge/performance-metrics.md` for exact formulas.

---

### Step 3: Scenario-Specific Deep-Dive Analysis
Categorize and evaluate system behavior according to scenario intent:
* **Load Scenario:** Baseline verification against target SLA/SLO.
* **Stress Scenario:** Saturation curve, inflection point where P95 degrades, and identification of write-contention or database lock bottlenecks.
* **Spike Scenario:** Peak surge resistance and speed of queue draining / recovery after load drops.
* **Endurance Scenario:** Time-window degradation trend (e.g., 3-minute segmented buckets) and memory ceiling stability.
* Consult `knowledge/result-analysis.md` for degradation detection heuristics.

---

### Step 4: Visual Resource Evidence Validation
1. Verify system hardware configuration and hostname against known environments.
2. Confirm CPU utilization percentage and memory RSS curves from concurrent `htop` / Task Manager screenshots.
3. Consult `knowledge/evidence-analysis.md` for visual audit rules.

---

### Step 5: Generate Standardized Analysis Report
Format the findings strictly following `templates/performance-analysis-template.md`.

---

## Output Artifacts
* Structured Performance Analysis Markdown Document: `performance_analysis_report.md`
