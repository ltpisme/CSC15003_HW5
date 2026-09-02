
# HW05 — Step 2: Performance Test Design — Load, Stress, Spike

> **Tài liệu:** `temp/AI/02_design.md`
> **Căn cứ:** `temp/AI/01_scope.md`, `ref/HW05.md`, `temp/Requirement/03_master.md`, `temp/Requirement/02_deliverable_submission.md`
> **Trạng thái:** Hoàn tất thiết kế — Chờ Human Review trước Step 3

---

## 1. Common E2E Workflow

HW05 yêu cầu 3 scenario **Load, Stress, Spike** sử dụng cùng một E2E workflow. Không tạo business workflow riêng cho từng scenario.

### 1.1 Workflow

```text
Register
   ↓
Login
   ↓
GET Products
   ↓
POST Cart
   ↓
GET Cart
   ↓
Forgot Password
```

### 1.2 API mapping

| Step | API                          | Role                                    |
| ---- | ---------------------------- | --------------------------------------- |
| 1    | POST`/api/register`        | Supporting — tạo account              |
| 2    | POST`/api/login`           | Supporting — lấy authentication token |
| 3    | GET`/api/products`         | Main — Read-heavy                      |
| 4    | POST`/api/cart`            | Main — Transactional                   |
| 5    | GET`/api/cart`             | Supporting — verify cart               |
| 6    | POST`/api/forgot-password` | Main — Auth-heavy                      |

**Không được thay đổi API selection khi tạo JMX.**

### 1.3 Correlation

JMX phải lấy dynamic values từ response thực tế theo API contract:

```text
Register → user/account data
Login → authentication token
Products → product ID + dữ liệu cần cho Add Cart
Cart → response data nếu workflow cần
Forgot Password → reset token nếu API trả về
```

JSONPath/field name phải được **xác minh từ API specification hoặc backend implementation**, không tự đoán.

---

## 2. Data-Driven Test Data

### 2.1 CSV

Chỉ tạo CSV cho dữ liệu thực sự được workflow sử dụng.

Khuyến nghị:

```text
jmeter/data/user_profiles.csv
```

Ví dụ:

```csv
name_prefix,base_password
Nguyen Van,Perf@Test2026!
Tran Thi,Perf@Test2026!
Le Van,Perf@Test2026!
```

Không thêm `shipping_address`, `phone` hoặc CSV khác nếu workflow không sử dụng.

### 2.2 JMeter CSV configuration

```text
File: data/user_profiles.csv
Variables: name_prefix,base_password
Delimiter: ,
Recycle on EOF: True
Stop thread on EOF: False
Sharing Mode: All threads
```

### 2.3 Unique account

Mỗi VU/iteration phải sử dụng account riêng, không dùng static account chung cho toàn bộ test.

Email phải chứa **run ID** để tránh collision giữa:

```text
Load
Stress
Spike
Rerun
```

Ví dụ logic:

```text
perf_<run_id>_<thread>_<iteration>@perf.eshop.vn
```

Uniqueness được thiết kế để tránh collision trong test run; không tuyên bố uniqueness tuyệt đối.

Password có thể lấy từ CSV và được reuse cho:

```text
Register → Login
```

Không regenerate password giữa các bước.

---

## 3. Common JMeter Test Flow

Cả 3 JMX phải giữ cùng business flow:

```text
Test Plan
└── Thread Group / workload controller
    ├── CSV Data Set Config
    ├── HTTP Request Defaults
    ├── HTTP Header Manager
    │
    └── E2E Workflow
        ├── POST /register
        │   └── extract required account data
        │
        ├── POST /login
        │   └── extract token
        │
        ├── GET /products
        │   └── extract product data
        │
        ├── POST /cart
        │   └── Authorization: Bearer ${token}
        │
        ├── GET /cart
        │   └── verify successful cart response
        │
        └── POST /forgot-password
            └── use the same registered email
```

### Assertions

Mỗi critical request phải có assertion phù hợp:

* expected HTTP status;
* required response field/message;
* required correlation value exists.

Assertions phải dựa trên actual API contract.

---

## 4. Workload Design

Ba scenario dùng **cùng workflow**, chỉ thay đổi workload profile.

Các VU/time values dưới đây là **initial test-design assumptions**, không phải production capacity hoặc measured performance. Chúng phải được ghi nhận là calibration targets và không được trình bày như kết quả thực tế.

| Parameter        |                               Load |                           Stress |                                    Spike |
| ---------------- | ---------------------------------: | -------------------------------: | ---------------------------------------: |
| Peak VUs         |                                 20 |                              100 |                                       80 |
| Pattern          | Smooth ramp → steady → ramp-down | Step-up → high load → recovery | Low baseline → sudden surge → recovery |
| Ramp-up          |                                30s |                         4 stages |                                10s surge |
| Steady/hold      |                               180s |                    35s per stage |                                 60s peak |
| Ramp-down        |                                15s |                              20s |                                      10s |
| Approx. duration |                               225s |                             300s |                                     170s |
| Think time       |                             ~500ms |                       100–300ms |                          ~100ms at spike |

### 4.1 Load

Purpose:

* establish normal-load behavior;
* measure latency, throughput and error rate;
* provide a reference for comparison with Stress and Spike.

Profile:

```text
0–30s       : ramp to 20 VUs
30–210s     : steady 20 VUs
210–225s    : ramp down
```

20 VUs là **initial calibration target**, không phải claim về production traffic.

---

### 4.2 Stress

Purpose:

* progressively increase concurrency;
* identify degradation/saturation;
* observe error rate, latency and backend resource behavior.

Target stages:

```text
25 VUs
50 VUs
75 VUs
100 VUs
```

Mỗi stage:

```text
~15s ramp
~35s hold
```

Sau stage cuối thực hiện ramp-down/recovery theo tổng thời gian thiết kế.

100 VUs chỉ là **test ceiling/initial stress target**. Không gọi đây là breaking point trước khi có kết quả thực tế.

Nếu hệ thống bắt đầu lỗi trước 100 VUs, ghi nhận mức lỗi/saturation thực tế thay vì cố tăng tải.

---

### 4.3 Spike

Purpose:

* evaluate response to sudden concurrency increase;
* measure behavior during peak;
* observe recovery after the spike.

Profile:

```text
0–30s       : 5 VUs baseline
30–40s      : increase 5 → 80 VUs
40–100s     : hold 80 VUs
100–110s    : decrease 80 → 5 VUs
110–170s    : recovery observation
```

80 VUs là **initial spike target**.

Việc tăng từ 5 → 80 VUs là 16× về concurrency target; **không được kết luận RPS cũng tăng chính xác 16×**. RPS thực tế phải lấy từ JMeter results.

---

## 5. Throughput and Calibration

Không sử dụng performance numbers chưa được đo như baseline thực tế.

Có thể dùng Little's Law để kiểm tra tính hợp lý của workload:

```text
N ≈ X × R
```

Trong đó:

```text
N = concurrent users
X = iteration throughput
R = average iteration cycle time
```

Với workflow 6 HTTP requests:

```text
HTTP request throughput ≈ iteration throughput × 6
```

Đây chỉ là **ước tính lý thuyết**. Throughput thực tế phải lấy từ JMeter.

### Calibration rule

Không hardcode các claim như:

```text
20ms response time
30 RPS
100 RPS
production capacity
breaking point
```

nếu chưa có evidence.

Sau calibration/official run, báo cáo phải phân biệt rõ:

```text
Design assumption
Measured result
Observed limit
Unknown
```

---

## 6. Think Time

Think time được dùng để tạo workload ổn định và tránh biến test thành request flood không chủ đích.

Target:

```text
Load  : ~500ms average
Stress: 100–300ms
Spike : ~100ms during spike
```

Nếu sử dụng Gaussian Random Timer, `mean` và `deviation` phải được ghi rõ; không mô tả Gaussian là phân phối bị giới hạn cứng trong một khoảng nếu JMeter không cấu hình giới hạn đó.

Think time là workload parameter và có thể được điều chỉnh trong calibration nếu cần.

---

## 7. Endurance Test

HW05 yêu cầu endurance trong khoảng **10–15 phút** nhưng không tạo scenario thứ tư.

Tái sử dụng **Load test plan** với CLI properties, ví dụ:

```text
-Jthreads=<calibrated VU>
-Jrampup=<ramp-up>
-Jduration=900
-Jrun_id=endurance
```

Ví dụ:

```bash
jmeter -n \
  -t jmeter/<StudentID>_Load_<YYYYMMDD>.jmx \
  -Jthreads=15 \
  -Jrampup=30 \
  -Jduration=900 \
  -Jrun_id=endurance \
  -l results/endurance.jtl \
  -e -o results/endurance_report/
```

Giá trị `15 VUs` chỉ là ví dụ; mức endurance chính thức phải được chọn sau calibration.

Theo dõi:

* throughput;
* response time/p95;
* error rate;
* backend CPU;
* backend memory;
* dấu hiệu degradation trong thời gian dài.

Không gọi endurance là scenario thứ tư.

---

## 8. Reports / Listeners

HW05 yêu cầu các scenario có report/listener/view khác nhau.

Thiết kế:

| Scenario | Distinct listener/view | Purpose                         |
| -------- | ---------------------- | ------------------------------- |
| Load     | View Results Tree      | Functional/debug evidence       |
| Stress   | Aggregate Report       | Detailed performance statistics |
| Spike    | Summary Report         | Summary of spike behavior       |

Official performance execution phải dùng **JMeter CLI/non-GUI**.

`View Results Tree` không dùng để đánh giá throughput của official load test; chỉ dùng khi cần debug/functional evidence.

JTL và HTML Dashboard được tạo từ CLI khi chạy chính thức.

---

## 9. Backend Resource Evidence

Khi chạy official scenarios, cần thu thập JMeter results cùng backend resource usage:

```text
JMeter:
- throughput
- response time
- p95
- error rate

Backend:
- CPU
- memory
- relevant process/resource usage
```

Resource evidence phải có timestamp/context đủ để đối chiếu với test run.

Không start/stop/restart/modify SUT trong quá trình thực hiện test.

---

## 10. Test Plan Naming

Tạo đúng 3 JMX:

```text
jmeter/<StudentID>_Load_<YYYYMMDD>.jmx
jmeter/<StudentID>_Stress_<YYYYMMDD>.jmx
jmeter/<StudentID>_Spike_<YYYYMMDD>.jmx
```

CSV đặt trong:

```text
jmeter/data/
```

Không dùng absolute machine-specific paths.

---

## 11. AI Initial Design → Human Review

AI-generated design phải được phân biệt với human correction.

Các điểm human review chính:

1. API contract phải đối chiếu backend/API specification.
2. Register → Login phải có authentication correlation đúng.
3. Product data phải được correlation trước khi Add Cart.
4. Account phải được isolate giữa VUs/runs.
5. Ba scenario phải dùng cùng E2E workflow.
6. Workload numbers là initial assumptions, không phải measured capacity.
7. Stress phải tìm saturation thay vì tuyên bố trước breaking point.
8. Spike phải phân biệt VU increase và actual RPS.
9. Endurance phải reuse Load plan, không tạo scenario thứ tư.
10. Official measurement phải dùng CLI và thu thập backend resource evidence.

### AI critique

AI không được tạo ra các "AI initial results" hoặc các claim về previous AI output nếu không có evidence từ quá trình làm việc.

Phần critique cuối cùng phải phản ánh:

```text
AI initial proposal
        ↓
Weakness identified
        ↓
Human decision/correction
        ↓
Final design
```

---

## 12. Step 3 Handoff

Step 3 được phép sử dụng tài liệu này để tạo JMX, nhưng phải:

* giữ nguyên API selection;
* giữ nguyên E2E workflow;
* verify API contract trước khi tạo sampler/extractor;
* tạo đúng 3 JMX;
* tạo CSV cần thiết;
* kiểm tra variable resolution;
* kiểm tra correlation;
* parse/validate JMX;
* chạy smoke test nhỏ nếu cần;
* không chạy official Load/Stress/Spike;
* ghi kết quả vào `temp/AI/03_validation.md`;
* dừng để Human Review.

**Không tự ý thêm API, workflow, scenario hoặc performance claim ngoài thiết kế này.**
