# HW05 Step 4 — Official Performance Execution Report

> **Tài liệu:** `temp/AI/04_execution.md`  
> **Nguồn căn cứ:** `ref/HW05.md`, `temp/AI/01_scope.md`, `temp/AI/02_design.md`, `temp/AI/03_validation.md`  
> **Trạng thái thực thi:** **HOÀN THÀNH 100% (4/4 Scenarios Executed & Verified)**  
> **Thời gian thực hiện:** 2026-09-02 15:05:00 → 15:45:00 UTC+7  

---

## Environment

- **SUT:** `http://localhost:3000` (Node.js Express EShop Backend + SQLite Database)
- **Hostname:** `ltp`
- **OS:** `Linux ltp 6.19.14-108.fc42.x86_64 #1 SMP PREEMPT_DYNAMIC Thu May 21 18:06:59 UTC 2026 x86_64 GNU/Linux` (Fedora Linux 42)
- **CPU:** `13th Gen Intel(R) Core(TM) i5-13420H` (12 vCPUs: 4 P-Cores + 4 E-Cores, 16 Threads / 12 logical processors detected)
- **RAM:** `15 GiB Physical RAM` (Available: ~6.5 GiB, Buff/Cache: 7.2 GiB)
- **JVM:** `OpenJDK Runtime Environment (Red_Hat-21.0.11.0.10-2) (build 21.0.11+10), 64-Bit Server VM`
- **JMeter:** `Apache JMeter 5.6.3` (`/opt/apache-jmeter-5.6.3/bin/jmeter`)

---

## Load

- **JMX:** `jmeter/23127452_Load_20260902.jmx`
- **Run ID:** `load`
- **Command:**
  ```bash
  /opt/apache-jmeter-5.6.3/bin/jmeter -n -t jmeter/23127452_Load_20260902.jmx -l results/load/raw.jtl -e -o results/load/html
  ```
- **Start:** `2026-09-02 15:05:28`
- **End:** `2026-09-02 15:09:12`
- **Configured workload:** 20 VUs, 30s ramp-up, 180s steady-state hold, 15s ramp-down (Total duration: 225s), Think Time Gaussian (500ms ± 200ms)
- **Execution status:** **PASS** (100% completed)
- **Raw JTL:** `results/load/raw.jtl` (1,185,216 bytes)
- **HTML:** `results/load/html/index.html` (Dashboard Generated)
- **Screenshot:** `evidence/load/load_jmeter_resource.png` (Captured at steady-state hold)
- **Total samples:** `8,290`
- **Success:** `8,290`
- **Failure:** `0`
- **Error rate:** `0.00%`
- **Throughput:** `37.10 samples/sec`
- **Average:** `6.21 ms`
- **Median:** `4 ms`
- **P90:** `10 ms`
- **P95:** `13 ms`
- **P99:** `41 ms`
- **Min:** `1 ms`
- **Max:** `77 ms`
- **E2E verification:**
  - `01_POST_Register`: 1,389 samples (Avg: 11.11ms, Med: 9ms, P95: 37ms, Min: 4ms, Max: 61ms)
  - `02_POST_Login`: 1,386 samples (Avg: 5.16ms, Med: 4ms, P95: 8ms, Min: 2ms, Max: 52ms)
  - `03_GET_Products_ReadHeavy`: 1,382 samples (Avg: 3.75ms, Med: 3ms, P95: 6ms, Min: 1ms, Max: 69ms)
  - `04_POST_Cart_Transactional`: 1,381 samples (Avg: 2.78ms, Med: 3ms, P95: 4ms, Min: 1ms, Max: 6ms)
  - `05_GET_Cart_Verify`: 1,378 samples (Avg: 2.44ms, Med: 2ms, P95: 4ms, Min: 1ms, Max: 6ms)
  - `06_POST_ForgotPassword_AuthHeavy`: 1,374 samples (Avg: 12.01ms, Med: 10ms, P95: 39ms, Min: 5ms, Max: 77ms)
  - *Correlation check:* Dynamic user email generated per VU iteration, JWT token extracted from `02_POST_Login` and successfully passed into `04_POST_Cart_Transactional` and `05_GET_Cart_Verify` via Bearer header. First product ID/name/price extracted from `03_GET_Products_ReadHeavy` and fed to Cart.
- **Resource observation:**
  - CPU usage observed at steady state: ~15–25% overall system CPU.
  - Node.js backend memory: ~60–80 MB RSS, stable throughout the run.
- **Issues:** None (0 HTTP errors, 0 Assertion failures).

---

## Stress

- **JMX:** `jmeter/23127452_Stress_20260902.jmx`
- **Run ID:** `stress`
- **Command:**
  ```bash
  /opt/apache-jmeter-5.6.3/bin/jmeter -n -t jmeter/23127452_Stress_20260902.jmx -l results/stress/raw.jtl -e -o results/stress/html
  ```
- **Start:** `2026-09-02 15:13:12`
- **End:** `2026-09-02 15:18:11`
- **Configured workload:** Staircase 4 stages (25 → 50 → 75 → 100 VUs), total duration 300s, Think Time Uniform (100–300ms)
- **Execution status:** **PASS** (100% completed)
- **Raw JTL:** `results/stress/raw.jtl` (10,122,096 bytes)
- **HTML:** `results/stress/html/index.html` (Dashboard Generated)
- **Screenshot:** `evidence/stress/stress_jmeter_resource.png` (Captured during high-load 75–100 VUs stage)
- **Total samples:** `64,231`
- **Success:** `64,231`
- **Failure:** `0`
- **Error rate:** `0.00%`
- **Throughput:** `215.03 samples/sec`
- **Average:** `140.16 ms`
- **Median:** `47 ms`
- **P90:** `323 ms`
- **P95:** `638 ms`
- **P99:** `1,640 ms`
- **Min:** `0 ms`
- **Max:** `3,234 ms`
- **E2E verification:**
  - `01_POST_Register`: 10,770 samples (Avg: 436.25ms, Med: 223ms, P90: 1201ms, P95: 1760ms, P99: 2301ms, Max: 3234ms)
  - `02_POST_Login`: 10,704 samples (Avg: 87.98ms, Med: 61ms, P90: 190ms, P95: 246ms, P99: 630ms, Max: 1375ms)
  - `03_GET_Products_ReadHeavy`: 10,697 samples (Avg: 81.20ms, Med: 56ms, P90: 181ms, P95: 227ms, P99: 525ms, Max: 1185ms)
  - `04_POST_Cart_Transactional`: 10,692 samples (Avg: 36.28ms, Med: 21ms, P90: 77ms, P95: 105ms, P99: 296ms, Max: 1030ms)
  - `05_GET_Cart_Verify`: 10,688 samples (Avg: 36.95ms, Med: 23ms, P90: 82ms, P95: 110ms, P99: 278ms, Max: 1061ms)
  - `06_POST_ForgotPassword_AuthHeavy`: 10,680 samples (Avg: 160.20ms, Med: 117ms, P90: 332ms, P95: 465ms, P99: 915ms, Max: 1637ms)
  - *Correlation check:* All 6 samplers chained without breaking correlation across 10,680+ completed full E2E iterations.
- **Resource observation:**
  - At 100 VUs peak, CPU usage reached ~65–80% on host CPU cores.
  - Latency of write-heavy and compute-heavy endpoints (`01_POST_Register` bcrypt hashing + SQLite write lock) increased significantly up to ~3.2s Max, showing the primary bottleneck under concurrency.
- **Issues:** No crash or process exit; 0 HTTP 5xx errors; SQLite handled transactions without returning `SQLITE_BUSY` error codes to client due to sequential internal queuing.

---

## Spike

- **JMX:** `jmeter/23127452_Spike_20260902.jmx`
- **Run ID:** `spike`
- **Command:**
  ```bash
  /opt/apache-jmeter-5.6.3/bin/jmeter -n -t jmeter/23127452_Spike_20260902.jmx -l results/spike/raw.jtl -e -o results/spike/html
  ```
- **Start:** `2026-09-02 15:20:49`
- **End:** `2026-09-02 15:23:37`
- **Configured workload:** Baseline 5 VUs (0–30s) → Surge 5 → 80 VUs (30–40s) → Peak hold 80 VUs (40–100s) → Ramp-down 80 → 5 VUs (100–110s) → Recovery 5 VUs (110–170s). Think Time Gaussian (100ms ± 50ms)
- **Execution status:** **PASS** (100% completed)
- **Raw JTL:** `results/spike/raw.jtl` (3,436,678 bytes)
- **HTML:** `results/spike/html/index.html` (Dashboard Generated)
- **Screenshot:** `evidence/spike/spike_jmeter_resource.png` (Captured at peak surge 80 VUs)
- **Total samples:** `21,470`
- **Success:** `21,470`
- **Failure:** `0`
- **Error rate:** `0.00%`
- **Throughput:** `127.27 samples/sec` (Overall average across baseline, peak surge, and recovery phases)
- **Average:** `167.69 ms`
- **Median:** `60 ms`
- **P90:** `432 ms`
- **P95:** `876 ms`
- **P99:** `1,759 ms`
- **Min:** `1 ms`
- **Max:** `2,892 ms`
- **E2E verification:**
  - `01_POST_Register`: 3,619 samples (Avg: 603.09ms, Med: 503ms, P90: 1263ms, P95: 1896ms, P99: 2211ms, Max: 2892ms)
  - `02_POST_Login`: 3,583 samples (Avg: 86.06ms, Med: 75ms, P90: 167ms, P95: 218ms, P99: 423ms, Max: 969ms)
  - `03_GET_Products_ReadHeavy`: 3,575 samples (Avg: 82.64ms, Med: 73ms, P90: 162ms, P95: 200ms, P99: 376ms, Max: 969ms)
  - `04_POST_Cart_Transactional`: 3,568 samples (Avg: 34.57ms, Med: 27ms, P90: 64ms, P95: 86ms, P99: 204ms, Max: 834ms)
  - `05_GET_Cart_Verify`: 3,566 samples (Avg: 35.96ms, Med: 27ms, P90: 70ms, P95: 93ms, P99: 219ms, Max: 857ms)
  - `06_POST_ForgotPassword_AuthHeavy`: 3,559 samples (Avg: 158.00ms, Med: 132ms, P90: 286ms, P95: 399ms, P99: 1033ms, Max: 1171ms)
- **Resource observation:**
  - Instantaneous spike to 80 VUs caused sharp jump in CPU usage (~60–75%) during peak hold.
  - Once load ramped down back to 5 VUs baseline at t=110s, system resource immediately recovered to normal idle levels (<5% CPU) and latency returned to single-digit milliseconds without residual queue blockage.
- **Issues:** None. Rapid recovery verified successfully.

---

## Endurance

- **Duration:** `898.67s (~15 phút / 900 giây)`
- **Workload:** 20 VUs sustained continuous load, Think Time Gaussian (500ms ± 200ms), Reusing `jmeter/23127452_Load_20260902.jmx` with CLI parameter overrides (`-Jthreads=20 -Jrampup=30 -Jduration=900 -Jrun_id=endurance`)
- **Raw JTL:** `results/endurance/raw.jtl` (4,965,595 bytes)
- **HTML:** `results/endurance/html/index.html` (Dashboard Generated)
- **Screenshot:** `evidence/endurance/endurance_jmeter_resource.png` (Captured during sustained window)
- **Total samples:** `34,677`
- **Success:** `34,677`
- **Failure:** `0`
- **Throughput:** `38.59 samples/sec`
- **Error rate:** `0.00%`
- **Average:** `7.86 ms`
- **Median:** `5 ms`
- **P90:** `15 ms`
- **P95:** `25 ms`
- **P99:** `45 ms`
- **Min:** `0 ms`
- **Max:** `412 ms`
- **Time Window Degradation Analysis (5 × 3-minute windows):**
  - **Window 1 (00–03 min):** 6,573 samples \| Throughput: 36.52 req/s \| Err: 0.00% \| Avg: 7.28 ms \| P95: 18 ms \| P99: 42 ms
  - **Window 2 (03–06 min):** 7,053 samples \| Throughput: 39.18 req/s \| Err: 0.00% \| Avg: 7.92 ms \| P95: 27 ms \| P99: 45 ms
  - **Window 3 (06–09 min):** 7,022 samples \| Throughput: 39.01 req/s \| Err: 0.00% \| Avg: 7.47 ms \| P95: 23 ms \| P99: 44 ms
  - **Window 4 (09–12 min):** 7,022 samples \| Throughput: 39.01 req/s \| Err: 0.00% \| Avg: 7.75 ms \| P95: 24 ms \| P99: 45 ms
  - **Window 5 (12–15 min):** 7,007 samples \| Throughput: 38.93 req/s \| Err: 0.00% \| Avg: 8.83 ms \| P95: 30 ms \| P99: 47 ms
- **Resource trend:**
  - Memory consumption of Node.js process remained virtually flat (~65–85 MB RSS).
  - CPU usage maintained steady at ~15–20% throughout the entire 15-minute window.
  - No memory leak, event-loop starvation, or database connection pool exhaustion detected.
- **Endurance threshold:**
  - **Sustained RPS / Throughput:** `38.59 samples/sec (~6.43 full E2E transactions/sec)`
  - **Stable memory boundary:** `~85 MB Node.js RSS` (no upward drift observed over 15 min)
  - **Observation window:** `900 seconds (15 minutes)`
  - **Error rate:** `0.00%`
  - **Observed P95 Latency:** `25 ms`
- **Threshold methodology:** Measured through empirical sustained non-GUI execution of 20 VUs with pacing over a 15-minute duration; verified through segmented 3-minute time-window stability analysis and continuous OS-level resource monitoring.
- **Threshold status:** **Established with concrete empirical data** (Maximum verified stable endurance baseline: **38.59 RPS with 0.00% errors and P95 <= 25ms**).

---

## Lockout / Reset Handling

- **Lockout / Reset required during runs:** **No**
- **Observed condition:** Each test iteration generated a unique runtime identity (`perf_${run_id}_${__threadNum}_${counter}@perf.eshop.vn`) using valid credentials from `user_profiles.csv`.
- **Status:** All login requests passed HTTP 200 with 0 authentication failures. No account lockout was triggered on the SUT during Load, Stress, Spike, or Endurance executions.

---

## Evidence Map

| Evidence Type | File Path | File Size | Purpose & Content |
| :--- | :--- | :--- | :--- |
| **Load Evidence** | `evidence/load/load_jmeter_resource.png` | 521 KB | JMeter CLI foreground steady-state execution + `htop` CPU/MEM monitoring |
| **Stress Evidence** | `evidence/stress/stress_jmeter_resource.png` | 552 KB | JMeter CLI at 75–100 VUs high-load stage + `htop` system resource utilization |
| **Spike Evidence** | `evidence/spike/spike_jmeter_resource.png` | 498 KB | JMeter CLI at 80 VUs peak surge + `htop` resource jump and recovery monitoring |
| **Endurance Evidence** | `evidence/endurance/endurance_jmeter_resource.png` | 616 KB | JMeter CLI 15-minute sustained soak test + `htop` memory stability verification |
| **Hardware Evidence** | `evidence/hardware/hardware_hostname.png` | 35 KB | System hardware report (`hostname`, `uname -a`, `lscpu`, `free -h`) |

---

## Problems / Limitations

1. **Write-contention bottleneck on `01_POST_Register`:**
   - Under 100 VUs (Stress) and 80 VUs (Spike), response time for registration spiked up to ~3.2 seconds (P95: 1.76s–1.90s). This is caused by bcrypt computational overhead combined with synchronous SQLite database write serialization.
2. **Read vs. Write latency disparity:**
   - Read-heavy endpoint (`GET /api/products`) and Transactional Cart endpoints (`POST /api/cart`, `GET /api/cart` in-memory storage) maintained low latencies (P95 <= 227ms in Stress, P95 <= 11ms in Endurance). Auth/Register database endpoints represent >80% of total E2E latency under heavy load.
3. **Hardware boundary:**
   - Local single-machine testing (JMeter client + Node.js SUT + SQLite on the same host) introduces minor resource competition between JMeter JVM and Node.js process during 100 VUs stress peak.
