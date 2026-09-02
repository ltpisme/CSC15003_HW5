# Visual Evidence & Resource Monitoring Audit

## 1. Principles of Visual Evidence Verification

### A. Non-Negotiable Rules
1. **Never Infer Numeric Latencies from Screenshots:** Pixel inspection cannot replace exact millisecond counts from `raw.jtl`.
2. **Corroborate Concurrency Windows:** Ensure the timestamp displayed in terminal windows matches the execution window of the corresponding JTL log file.
3. **Verify Attributable Hardware Identity:** Validate that the hostname and processor specifications shown in terminal screenshots correspond to the verified test environment.

---

## 2. Resource Monitor Screenshot Audit Checklist
* **Foreground Execution:** Screenshot shows JMeter running in CLI or displaying summary progress.
* **Target Process Filter:** Resource monitor (`htop`, `top`, Task Manager) explicitly filters or highlights the SUT backend process ID (PID).
* **CPU Core Distribution:** Check whether load is distributed across multiple vCPUs or bottlenecked on a single core (e.g., single-threaded Node.js event loop).
* **Memory State (RSS):** Verify that resident memory stays within physical limits and does not trigger OS swap activity.
