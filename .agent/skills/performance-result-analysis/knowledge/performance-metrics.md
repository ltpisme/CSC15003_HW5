# Performance Result Metrics & Statistical Definitions

## 1. Statistical Aggregation Reference

### A. Mathematical Definitions
* **Sample Count ($N$):** The exact integer count of HTTP requests completed during the test window.
* **Error Rate ($E$):**
  $$E = \frac{\text{Count}(\text{success} = \text{false})}{N} \times 100\%$$
* **Throughput / RPS ($T$):**
  $$T = \frac{N}{t_{\text{end}} - t_{\text{start}}} \quad (\text{samples/second})$$
* **Arithmetic Mean Latency ($\bar{x}$):**
  $$\bar{x} = \frac{1}{N} \sum_{i=1}^N x_i$$
* **Percentile ($P_k$):** The response time value below which $k\%$ of observations fall.
  - $P_{50}$ (Median): Robust central tendency measure, unaffected by extreme outliers.
  - $P_{90}$ / $P_{95}$: Service Level Agreement standard benchmark.
  - $P_{99}$: Tail latency benchmark measuring edge-case user experience.

---

## 2. Common Metric Interpretation Errors
1. **The Average Fallacy:** Reporting only Arithmetic Mean while ignoring P95/P99. A system with a 10ms average can experience 2-second P99 spikes.
2. **Percentile Interpolation Hallucination:** Estimating percentiles from small sample windows rather than sorting the full dataset.
3. **Throughput vs. Concurrency Confusion:** Believing that doubling Virtual Users always doubles throughput. When saturation occurs, adding VUs increases latency while throughput plateaus or drops.
