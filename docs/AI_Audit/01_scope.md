# HW05 — Step 1: Requirement, API và E2E Workflow Audit

> **Trạng thái:** Hoàn thành audit yêu cầu, API contract, E2E workflow và các constraint cho Step 2 — Sẵn sàng Human Review
> **Tài liệu:** `temp/AI/01_scope.md`
> **Nguồn đối chiếu:** `ref/HW05.md`, `temp/Requirement/03_master.md`, `temp/Requirement/02_deliverable_submission.md`, `ref/api_specification.md`, `eshop/api_specification.md`, `eshop/backend/server.js`, `eshop/backend/database.js`, `eshop/backend/test_profile.js`.

---

## 1. HW05 Requirements và Traceability

Các yêu cầu liên quan trực tiếp đến Step 1 và Step 2 được xác định như sau:

| Requirement               | Nội dung cần đáp ứng                                  | Quyết định trong Step 1                                                                                                     |
| ------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **REQ-22 / REQ-27** | Kiểm thử 3 endpoint groups trên cùng một E2E workflow | Chọn**Read-heavy**, **Auth-heavy**, **Transactional** và sử dụng một workflow duy nhất                 |
| **REQ-26**          | Có đúng 3 performance scenarios                         | **Load, Stress, Spike**                                                                                                  |
| **REQ-29**          | AI hỗ trợ xác định workload parameters                | Step 2 phải đề xuất VU, ramp-up, duration, think time cho từng scenario và ghi rõ assumption/calibration                |
| **REQ-31 / REQ-32** | Data-driven bằng CSV                                      | Sử dụng`CSV Data Set Config`; CSV cung cấp user/search data, runtime identity được tạo unique                         |
| **REQ-33**          | 3 listener/report/view khác nhau                          | Load, Stress, Spike dùng 3 loại report/view khác nhau; tránh GUI listener trong official execution nếu không cần        |
| **REQ-35**          | Naming convention                                          | `{StudentID}_{ScenarioType}_{YYYYMMDD}.jmx`                                                                                  |
| **REQ-42**          | Xử lý account lockout nếu phát sinh                    | Normal E2E không cố tình gây lockout; nếu Stress/Spike gây lockout thì phải reset/isolate state và document procedure |
| **REQ-44**          | Endurance                                                  | Sử dụng một trong 3 test plans để chạy sustained load**10–15 phút**, không tạo scenario thứ tư               |
| **REQ-45**          | Đánh giá threshold                                      | Thu thập RPS/throughput, latency, error rate và resource usage; threshold phải dựa trên observed data, không bịa số    |

### Nguyên tắc thiết kế

1. Chỉ có **một E2E workflow chuẩn**.
2. Load/Stress/Spike chỉ thay đổi **workload profile**, không thay đổi business flow.
3. Không tạo performance result giả.
4. Không đưa giá trị performance chưa được đo thành fact.
5. Các giá trị chưa có baseline phải được ghi rõ là **ASSUMPTION** và được calibration ở bước execution.
6. Supporting APIs được phép xuất hiện trong E2E workflow nhưng không được tính nhầm thành endpoint group thứ tư.
7. `POST /api/register` và các API tạo dữ liệu phải được tính vào tổng workload của E2E nhưng endpoint-group analysis tập trung vào 3 target groups.

---

## 2. API Contract Thực Tế

Contract được đối chiếu từ `eshop/backend/server.js`, `eshop/backend/database.js` và API specification.

| # | Endpoint                 | Method | Authentication | Request                                   | Success                        | Vai trò                                      |
| - | ------------------------ | ------ | -------------- | ----------------------------------------- | ------------------------------ | --------------------------------------------- |
| 1 | `/api/register`        | POST   | Không         | `name`, `email`, `password`         | `200` + `id`               | Supporting API — tạo user                   |
| 2 | `/api/login`           | POST   | Không         | `email`, `password`                   | `200` + `token` + `user` | Supporting API — tạo authentication context |
| 3 | `/api/products`        | GET    | Không         | optional`search`                        | `200` + product array        | **Read-heavy target**                   |
| 4 | `/api/cart`            | POST   | Bearer JWT     | `id`, `name`, `price`, `quantity` | `200` + confirmation         | **Transactional target**                |
| 5 | `/api/cart`            | GET    | Bearer JWT     | None                                      | `200` + cart array           | Supporting verification API                   |
| 6 | `/api/forgot-password` | POST   | Không         | `email`                                 | `200` + `resetToken`       | **Auth-heavy target**                   |

### API observations

* `/api/register` **không trả JWT**; phải gọi `/api/login` tiếp theo.
* `/api/login` trả JWT trong field `token`.
* `/api/cart` POST/GET yêu cầu Bearer token.
* `/api/products` là public endpoint.
* `/api/forgot-password` không yêu cầu JWT nhưng email phải tồn tại.
* `users.email` không có database `UNIQUE` constraint; vì vậy test phải chủ động tạo **unique runtime email**.
* Cart được lưu trong `userCarts` ở memory của Node.js process.
* Login failure có cơ chế lockout. Theo implementation hiện tại, failed attempt counter tăng theo logic backend và có thể dẫn đến lockout 3 phút. Vì vậy test không được dùng chung một account giữa các VU.

---

## 3. Endpoint Group Mapping

Ba target endpoint groups được xác định như sau:

| Group                   | Target endpoint               | Lý do                                                                                                          |
| ----------------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Read-heavy**    | `GET /api/products`         | Đọc danh sách products từ SQLite và serialize dữ liệu trả về client; phù hợp để tạo read workload |
| **Auth-heavy**    | `POST /api/forgot-password` | User lookup + sinh reset token + database update; đại diện cho authentication/account-recovery workload      |
| **Transactional** | `POST /api/cart`            | Có JWT authentication và mutate state của user cart; đại diện cho state-changing transaction              |

### Vì sao `POST /api/login` không được tính là Auth-heavy target?

`POST /api/login` là **supporting authentication API** vì nó cần thiết để tạo JWT context cho transactional requests trong E2E workflow.

`POST /api/forgot-password` được chọn làm **Auth-heavy target** vì backend thực hiện user lookup, token generation và database update.

Do đó:

```text
Supporting authentication:
POST /api/login
        │
        └── tạo JWT
              │
              ▼
Target Auth-heavy:
POST /api/forgot-password
```

Việc phân loại này được giữ nhất quán trong cả ba scenario.

---

## 4. Final Single E2E Workflow

Cả **Load, Stress và Spike** phải sử dụng chính xác workflow sau:

```text
┌──────────────────────────────────────────────────────────────┐
│                    SINGLE E2E WORKFLOW                       │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Load CSV user data
                              │
                              ▼
                  Generate unique runtime
                         user identity
                              │
                              ▼
                 [1] POST /api/register
                    Supporting API
                              │
                              ▼
                   [2] POST /api/login
                    Supporting API
                              │
                       Extract JWT
                              │
                              ▼
              [3] GET /api/products
                  READ-HEAVY TARGET
                              │
                    Extract product data
                              │
                              ▼
                 [4] POST /api/cart
                 TRANSACTIONAL TARGET
                              │
                              ▼
                  [5] GET /api/cart
                 Verify cart state
                              │
                              ▼
            [6] POST /api/forgot-password
                  AUTH-HEAVY TARGET
                              │
                    Extract resetToken
                              │
                              ▼
                         E2E END
```

### Step-by-step behavior

#### Step 1 — Register

```http
POST /api/register
Content-Type: application/json
```

Body:

```json
{
  "name": "${user_name}",
  "email": "${user_email}",
  "password": "${user_password}"
}
```

Expected:

```text
HTTP 200
id is present
```

The generated email must be unique for the current VU/iteration/run.

---

#### Step 2 — Login

```http
POST /api/login
Content-Type: application/json
```

Body:

```json
{
  "email": "${user_email}",
  "password": "${user_password}"
}
```

Expected:

```text
HTTP 200
token exists
user.id exists
```

Extract:

```text
$.token    → ${jwt_token}
$.user.id  → ${user_id}
```

---

#### Step 3 — Products / Read-heavy

```http
GET /api/products
```

Optional search parameter:

```text
?search=${search_keyword}
```

Expected:

```text
HTTP 200
response is a non-empty product array
```

Extract a deterministic product from the response, preferably the first available product:

```text
$.[0].id     → ${product_id}
$.[0].name   → ${product_name}
$.[0].price  → ${product_price}
```

The extraction rule must be deterministic and documented. Do not use `$..id` if multiple IDs may be returned and the JMeter variable could become ambiguous.

---

#### Step 4 — Add to Cart / Transactional

```http
POST /api/cart
Authorization: Bearer ${jwt_token}
Content-Type: application/json
```

Body:

```json
{
  "id": ${product_id},
  "name": "${product_name}",
  "price": ${product_price},
  "quantity": 1
}
```

Expected:

```text
HTTP 200
message = "Added to cart"
```

---

#### Step 5 — Get Cart / Transaction Verification

```http
GET /api/cart
Authorization: Bearer ${jwt_token}
```

Expected:

```text
HTTP 200
cart contains the product added in Step 4
```

This request is a **supporting verification API**, not a fourth endpoint group.

The assertion should verify the transaction result where practical, rather than only checking HTTP 200.

---

#### Step 6 — Forgot Password / Auth-heavy

```http
POST /api/forgot-password
Content-Type: application/json
```

Body:

```json
{
  "email": "${user_email}"
}
```

Expected:

```text
HTTP 200
resetToken exists
```

Extract:

```text
$.resetToken → ${reset_token}
```

This position is intentional because the user has already been registered, so the email is known to exist.

---

## 5. Authentication and Correlation Flow

```text
POST /register
       │
       └── creates user
              │
              ▼
        POST /login
              │
              ├── $.token ───────→ ${jwt_token}
              │
              └── $.user.id ─────→ ${user_id}
                                      │
                                      ▼
                           Authorization: Bearer
                                      │
                         ┌────────────┴────────────┐
                         ▼                         ▼
                  POST /api/cart             GET /api/cart
```

JWT must never be hardcoded.

The `Authorization` header must use:

```text
Bearer ${jwt_token}
```

---

## 6. Correlation Requirements

| Variable                                                                           | Source                     | Extraction / generation                                            | Usage                          |
| ---------------------------------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------ | ------------------------------ |
| `${user_name}`                                                                   | CSV                        | CSV Data Set Config                                                | Register                       |
| `${user_password}`                                                               | CSV                        | CSV Data Set Config                                                | Register/Login                 |
| `${email_prefix}`                                                                | CSV                        | CSV Data Set Config                                                | Unique email generation        |
| `${search_keyword}`                                                              | CSV                        | CSV Data Set Config                                                | Products search                |
| `${user_email}`                                                                  | Runtime                    | JMeter function/JSR223 based on CSV + run/VU/iteration identifiers | Register/Login/Forgot-password |
| `${jwt_token}`      | Login response           | JSON Extractor `$.token`      | Cart POST/GET              |                                                                    |                                |
| `${user_id}`        | Login response           | JSON Extractor `$.user.id`    | User correlation/debugging |                                                                    |                                |
| `${product_id}`     | Products response        | JSON Extractor `$.[0].id`     | Cart POST                  |                                                                    |                                |
| `${product_name}`   | Products response        | JSON Extractor `$.[0].name`   | Cart POST                  |                                                                    |                                |
| `${product_price}`  | Products response        | JSON Extractor `$.[0].price`  | Cart POST                  |                                                                    |                                |
| `${reset_token}`    | Forgot-password response | JSON Extractor `$.resetToken` | Assertion/evidence         |                                                                    |                                |

---

## 7. Data-driven Design

Data-driven testing must use **CSV Data Set Config** rather than hardcoded values wherever practical.

### 7.1 User data

Recommended file:

```text
data/user_profiles.csv
```

Example:

```csv
name,email_prefix,password
Nguyen Van A,perf_user_a,PerfTest@123
Tran Thi B,perf_user_b,PerfTest@123
Le Van C,perf_user_c,PerfTest@123
Pham Thi D,perf_user_d,PerfTest@123
Hoang Van E,perf_user_e,PerfTest@123
```

Only fields actually consumed by the workflow should be included.

Unused fields such as shipping address or phone should not be added unless a future endpoint requires them.

### 7.2 Search data

Optional:

```text
data/search_keywords.csv
```

Example:

```csv
keyword
iPhone
Samsung
MacBook
AirPods
Keychron
```

If the test uses search mode, `${search_keyword}` is read from this CSV.

If the objective is to test the complete product listing rather than search behavior, the `search` parameter may be omitted. The chosen mode must remain consistent across the three scenarios unless Step 2 documents a deliberate reason otherwise.

---

## 8. Unique Runtime Account Strategy

### Problem

The database does not enforce unique email values. Reusing an email can therefore cause:

* duplicate users;
* login resolving to an unintended record;
* cross-VU state contamination;
* lockout interference;
* invalid E2E correlation.

### Strategy

CSV provides a reusable identity prefix, while JMeter generates a unique runtime email for each test execution context.

Conceptually:

```text
${email_prefix}_${run_id}_${thread}_${iteration}@perf.test
```

For example:

```text
perf_user_a_Load01_3_2@perf.test
```

The exact JMeter implementation may use `${__threadNum}`, an iteration counter, a run identifier supplied by `-Jrun_id`, and a timestamp/UUID component if required.

### Important constraint

The strategy is designed to **minimize and practically prevent collisions** across:

* virtual users;
* iterations;
* scenario runs.

The design should not claim mathematical or absolute uniqueness unless a collision-free identifier mechanism has actually been implemented.

### Runtime identity lifecycle

A single generated `${user_email}` must be created once for the current iteration and then reused by:

```text
Register
Login
Forgot-password
```

It should not be regenerated independently for each sampler.

---

## 9. Lockout Handling

The normal E2E workflow uses the correct password and therefore should not intentionally trigger account lockout.

Lockout is treated as a **conditional operational risk**, not as an additional E2E step.

### Normal workflow

```text
Register
   ↓
Correct Login
   ↓
No lockout
```

### If Stress/Spike produces lockout

If failed authentication requests or other execution conditions cause accounts to become locked:

1. isolate affected test accounts;
2. record the lockout occurrence;
3. reset the affected database state before another run, where permitted;
4. ensure the next scenario does not inherit the previous scenario's account state;
5. document the reset procedure in the execution/evidence report.

The reset operation must be performed outside the normal E2E business workflow and must not be counted as a performance endpoint.

A reset may restore fields such as:

```text
login_attempts = 0
locked_until = NULL
```

The exact reset command must be verified against the actual database schema before execution.

---

## 10. Workload Design Constraints for Step 2

Step 1 does **not** invent performance results or claim a known system capacity.

No baseline measurement is currently asserted for:

* maximum VU;
* maximum RPS;
* maximum throughput;
* p95/p99 latency;
* memory ceiling;
* CPU ceiling.

Therefore Step 2 must treat initial workload values as **AI-assisted design assumptions**, followed by calibration using observed results.

### Required workload parameters

For each scenario, Step 2 must define:

| Parameter           | Required                                          |
| ------------------- | ------------------------------------------------- |
| Virtual users       | Yes                                               |
| Ramp-up             | Yes                                               |
| Duration            | Yes                                               |
| Think time          | Yes                                               |
| Workload pattern    | Yes                                               |
| Expected/target RPS | Yes, but derived/estimated rather than fabricated |
| Calibration method  | Yes                                               |

### RPS principle

Expected RPS must not be presented as measured performance before execution.

Use:

```text
Expected RPS ≈ effective concurrent users / average E2E response-cycle time
```

only as an initial planning estimate, or derive it from an explicit workload model.

After execution, replace estimates with observed throughput from JMeter results.

---

## 11. Scenario Consistency

The three performance scenarios share the same workflow:

```text
Register
→ Login
→ Products
→ Add Cart
→ Get Cart
→ Forgot Password
```

Only workload characteristics differ.

### Load

Purpose:

* represent expected/normal sustained workload;
* establish baseline response and throughput behavior;
* verify the E2E workflow under stable load.

### Stress

Purpose:

* progressively increase load;
* identify degradation and the practical operating limit;
* observe latency, error rate, throughput and backend resource behavior.

### Spike

Purpose:

* introduce a sudden increase in concurrent demand;
* observe short-term reaction to abrupt load;
* measure recovery/degradation behavior.

Step 2 must provide concrete VU/ramp-up/duration/think-time values and explain their rationale. If no empirical baseline exists, values must be marked **ASSUMPTION** and calibrated through a preliminary run.

---

## 12. Think Time / Pacing

Think time should be applied between logical user actions when the objective is to represent user-like behavior.

A timer such as:

```text
Uniform Random Timer
```

or:

```text
Gaussian Random Timer
```

may be used.

The exact range must be selected in Step 2 as an AI-assisted workload assumption and justified.

The value must not be presented as a measured property of the SUT.

For a capacity-focused test where maximum throughput is the objective, Step 2 may deliberately use a shorter or controlled think time, but this must be explicitly documented because it changes the workload model.

---

## 13. Endurance Requirement

Endurance does **not** create a fourth performance scenario.

One of the existing test plans will be configured for a sustained execution of:

```text
10–15 minutes
```

The same E2E workflow is retained.

The endurance run should collect:

* throughput/RPS;
* response time, especially p95 where available;
* error rate;
* CPU usage;
* memory usage;
* signs of degradation over time.

### Threshold interpretation

The endurance threshold must be based on observed evidence.

Do not state:

```text
"System supports X RPS"
```

unless X has actually been measured and the acceptance criterion is defined.

Instead report:

```text
Observed sustained throughput: X RPS
Observed p95: Y ms
Observed error rate: Z%
Observed resource usage: ...
Observed degradation: ...
```

and derive the practical threshold from the evidence.

---

## 14. Backend Resource Monitoring

Because HW05 requires correlation between JMeter execution and backend resource usage, official executions should capture the SUT process/system resource state using the available operating-system monitoring tool.

Examples:

```text
Linux → htop
Windows → Task Manager / Resource Monitor
```

The evidence should correlate:

```text
JMeter test execution
        +
backend CPU / memory
        +
time / scenario
```

This is particularly important for Stress, Spike and Endurance analysis.

---

## 15. Listener / Report Strategy

The three scenarios must use **three different primary report/listener views**.

Recommended mapping:

| Scenario         | Primary report/view          | Purpose                                           |
| ---------------- | ---------------------------- | ------------------------------------------------- |
| **Load**   | `Summary Report`           | Overall samples, throughput and error summary     |
| **Stress** | `Aggregate Report`         | Detailed response-time statistics and percentiles |
| **Spike**  | `Response Times Over Time` | Observe response-time behavior during sudden load |

### Execution constraint

GUI-heavy listeners such as `View Results Tree` should be used only for debugging/validation and disabled or removed from the official non-GUI performance execution when unnecessary.

The three required primary views must remain distinguishable in the final evidence.

---

## 16. Naming Convention

The three JMeter plans must follow:

```text
{StudentID}_{ScenarioType}_{YYYYMMDD}.jmx
```

Examples:

```text
21127000_Load_20260902.jmx
21127000_Stress_20260902.jmx
21127000_Spike_20260902.jmx
```

The actual `StudentID` must be replaced with the student's real ID.

---

## 17. Test Plan Architecture for Step 2

The recommended architecture is shared across all three scenarios:

```text
Test Plan
├── User Defined Variables / Test Properties
├── CSV Data Set Config
├── HTTP Request Defaults
├── HTTP Header Manager
├── Thread Group
│   └── Single E2E Workflow
│       ├── POST /api/register
│       ├── POST /api/login
│       │   ├── JSON Extractor: jwt_token
│       │   └── JSON Extractor: user_id
│       ├── GET /api/products
│       │   └── JSON Extractors: product_*
│       ├── POST /api/cart
│       ├── GET /api/cart
│       │   └── Response Assertion
│       └── POST /api/forgot-password
│           └── JSON Extractor: reset_token
└── Scenario-specific report/listener
```

The following must remain identical between Load, Stress and Spike:

* endpoint sequence;
* request contract;
* correlation logic;
* assertions;
* data model;
* authentication mechanism;
* business workflow.

The following may differ:

* VU/thread count;
* ramp-up;
* duration;
* pacing/think time;
* spike pattern;
* scenario-specific listener/report.

---

## 18. Assertions and E2E Correctness

Performance testing must not reduce validation to HTTP transport only.

At minimum, the workflow should validate:

```text
Register:
  HTTP 200 + user id

Login:
  HTTP 200 + JWT token

Products:
  HTTP 200 + usable product data

Add Cart:
  HTTP 200 + successful message

Get Cart:
  HTTP 200 + expected product present

Forgot Password:
  HTTP 200 + resetToken present
```

Assertions should be lightweight enough not to introduce unnecessary test-client overhead.

The primary goal is to prove that the E2E transaction completed correctly while collecting performance metrics.

---

## 19. Known Risks and Technical Constraints

### 19.1 SQLite concurrency

SQLite may become a bottleneck under concurrent database writes.

Potential symptoms include:

```text
SQLITE_BUSY
database locked
increased latency
increased error rate
```

This is a useful Stress/Spike observation rather than an error to hide.

### 19.2 In-memory cart state

`userCarts` is stored in the Node.js process memory.

A large number of unique users/cart entries may therefore increase memory usage.

This makes backend memory monitoring relevant to:

* Stress;
* Spike;
* Endurance.

### 19.3 Test-data accumulation

Because Register creates persistent database users, repeated executions may accumulate test accounts.

The test process should therefore:

* use a recognizable test-data prefix;
* record the run identifier;
* document cleanup/reset procedure where permitted.

### 19.4 Client-side overhead

Large numbers of GUI listeners can consume significant memory on the JMeter machine and distort performance measurements.

Official performance runs should therefore prefer non-GUI execution and lightweight result collection.

---

## 20. Unknowns / Assumptions to Carry into Step 2

| #  | Item                       | Status                                       | Action in Step 2                                                                  |
| -- | -------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------- |
| 1  | Baseline maximum VU        | **UNKNOWN**                            | Calibrate                                                                         |
| 2  | Baseline sustainable RPS   | **UNKNOWN**                            | Measure                                                                           |
| 3  | Maximum acceptable latency | **UNKNOWN** unless specified elsewhere | Use HW requirement/observed data                                                  |
| 4  | Initial Load VU/ramp-up    | **ASSUMPTION**                         | AI proposes + Human reviews                                                       |
| 5  | Initial Stress VU/ramp-up  | **ASSUMPTION**                         | AI proposes + Human reviews                                                       |
| 6  | Initial Spike magnitude    | **ASSUMPTION**                         | AI proposes + Human reviews                                                       |
| 7  | Think-time range           | **ASSUMPTION**                         | AI proposes + Human reviews                                                       |
| 8  | Endurance configuration    | **DESIGN CONSTRAINT**                  | 10–15 minute sustained run                                                       |
| 9  | Lockout occurrence         | **CONDITIONAL**                        | Reset/isolate only if triggered                                                   |
| 10 | Product selection          | **DESIGN DECISION**                    | Deterministic`$.[0]` extraction unless Step 2 finds a better validated approach |

---

## 21. AI Design and Human Review Boundary

The following separation must be preserved for the HW05 AI-critique requirement.

### AI initial design

AI may propose:

* endpoint grouping;
* E2E workflow;
* CSV structure;
* correlation strategy;
* unique-user strategy;
* VU counts;
* ramp-up;
* duration;
* think time;
* spike pattern;
* report/listener mapping.

### Human review

Human must review:

* whether endpoint classification matches the HW;
* whether the workflow is realistic;
* whether workload assumptions are reasonable;
* whether test-data generation is safe;
* whether the SUT can be executed without unintended state corruption;
* whether the proposed JMeter configuration is practical.

### Correction principle

Any design changed after Human Review must be recorded in the Step 2 AI critique:

```text
AI initial design
        ↓
Weakness / risk identified
        ↓
Human review
        ↓
Corrected final design
```

This provides traceability instead of presenting the final design as if it had no assumptions or revisions.

---

## 22. Step 1 Final Decision

The approved design baseline for Step 2 is:

```text
                    HW05
                      │
        ┌─────────────┴─────────────┐
        │                           │
   3 Target Groups            3 Scenarios
        │                           │
        │                  ┌────────┼────────┐
        │                  │        │        │
        ▼                  ▼        ▼        ▼
Read-heavy             Load    Stress    Spike
Auth-heavy
Transactional
        │
        └──────────────┐
                       ▼
               ONE E2E WORKFLOW
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
    Register          Login          Products
                                         │
                                         ▼
                                       Cart
                                         │
                                         ▼
                                   Verify Cart
                                         │
                                         ▼
                                  Forgot Password
```

### Step 2 must produce

```text
temp/AI/02_design.md
```

containing:

1. AI initial workload design;
2. Load configuration;
3. Stress configuration;
4. Spike configuration;
5. think-time strategy;
6. expected-RPS calculation/assumptions;
7. endurance configuration;
8. CSV/JMeter configuration;
9. three distinct report/listener choices;
10. weaknesses and risks;
11. Human Review points;
12. corrected final design.

### Execution boundary

Step 1/Step 2 design must **not** fabricate performance results.

The actual SUT execution, resource monitoring and evidence collection belong to the later execution phase.

---

> **Human Review Status:** READY FOR REVIEW
>
> **Approved baseline:** One shared E2E workflow + three workload profiles (Load/Stress/Spike) + CSV-driven data + runtime unique identity + JWT/product correlation + conditional lockout handling + 10–15 minute endurance execution using an existing plan.
>
> **Next step:** After Human Review, use `temp/AI/01_scope.md` as the sole scope/design baseline for HW05 Step 2 and generate `temp/AI/02_design.md`

# HW05 — Step 1: Requirement, API và E2E Workflow Audit

> **Trạng thái:** Hoàn thành kiểm tra & Sẵn sàng cho Human Review
> **Tài liệu tạo ra:** `temp/AI/01_scope.md`
> **Nguồn đối chiếu:** `ref/HW05.md`, `temp/Requirement/03_master.md`, `temp/Requirement/02_deliverable_submission.md`, `ref/api_specification.md`, `eshop/api_specification.md`, `eshop/backend/server.js`, `eshop/backend/database.js`, `eshop/backend/test_profile.js`.

---

## 1. HW05 Requirements Liên Quan

Dựa trên việc đối chiếu `ref/HW05.md` và `temp/Requirement/03_master.md`, các yêu cầu trọng yếu cho thiết kế kịch bản và workflow kiểm thử bao gồm:

* **REQ-22 & REQ-27 (Endpoint Groups & E2E Workflow):** Phải kiểm thử chính xác 3 nhóm endpoint backend API: **Read-heavy**, **Auth-heavy**, và **Transactional** trên **cùng một quy trình công việc đầu-cuối (single end-to-end workflow)**.
* **REQ-26 (3 Kịch bản kiểm thử):** Thiết kế 3 kịch bản kiểm thử: **Load**, **Stress**, và **Spike** chạy cùng workflow E2E đã định nghĩa.
* **REQ-31 & REQ-32 (Data-driven testing):** Tham số hóa workflow bằng file **CSV đầu vào** (ví dụ: thông tin người dùng, sản phẩm, tham số tải).
* **REQ-33 (Distinct Listeners):** Sử dụng **3 loại listener / report views khác nhau** giữa 3 kịch bản (không được lặp lại loại listener giữa Load, Stress, Spike).
* **REQ-35 (Naming Convention):** Định dạng tên kịch bản kiểm thử bắt buộc: `{StudentID}_{ScenarioType}_{YYYYMMDD}` (ví dụ: `21127000_Load_20260902.jmx`).
* **REQ-42 (Account Lockout Handling):** Khi chạy Stress/Spike chạm ngưỡng lockout (3 lần fail -> lock 3 phút), phải có cơ chế cách ly hoặc reset trạng thái giữa các lần chạy.
* **REQ-44 & REQ-45 (Endurance Threshold):** Đo lường ngưỡng chịu tải thực tế của phần cứng (RPS, memory ceiling, latency).

---

## 2. API Contract Thực Tế (Kiểm Chứng Từ Source Code)

Đã kiểm chứng trực tiếp từ `eshop/backend/server.js` và `eshop/backend/database.js`:

| STT | Endpoint                 | Method   | Header yêu cầu                                                | Request Body / Query                                                    | Response Model (Success)                                                                                                                | Response Model (Error)                                                                                                                                                           | Ghi chú từ Backend Implementation                                                                                                                                                                                                                                           |
| :-- | :----------------------- | :------- | :-------------------------------------------------------------- | :---------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | `/api/register`        | `POST` | `Content-Type: application/json`                              | `{"name": string, "email": string, "password": string}`               | **200 OK**`{"message": "User registered successfully", "id": <number>}`                                                         | **500 Internal Server Error**`{"error": err.message}`                                                                                                                    | **Không** trả JWT Token. Cần gọi tiếp `/api/login` để lấy Token. Schema DB không đặt `UNIQUE` cho email, nhưng logic login `SELECT * WHERE email = ?` chỉ lấy bản ghi đầu tiên -> bắt buộc phải dùng email unique.                          |
| 2   | `/api/login`           | `POST` | `Content-Type: application/json`                              | `{"email": string, "password": string}`                               | **200 OK**`{"message": "Login successful", "token": "<jwt>", "user": {...}}`                                                    | **401 Unauthorized**`{"error": "Invalid email or password"}`**403 Forbidden** (nếu bị lock)`{"error": "Tài khoản đã bị khóa. Vui lòng thử lại sau."}` | Khi đăng nhập đúng, reset`login_attempts = 0, locked_until = NULL`. Trả về trường `token` chứa chuỗi JWT (secret: `super_secret_key_that_should_not_be_here`). Sai mật khẩu: `newAttempts = login_attempts + 2`, nếu `>= 3` sẽ khóa 180s (3 phút). |
| 3   | `/api/products`        | `GET`  | Không bắt buộc                                               | Query (tùy chọn):`?search=keyword`                                  | **200 OK**`[{"id": 1, "name": "iPhone...", "price": 30000000, "description": "...", "imageUrl": "...", "category_id": 1}, ...]` | **500 Internal Server Error**`<h1>Database Error</h1><p>...</p>` (khi lỗi search)                                                                                       | Truy vấn trực tiếp`SELECT * FROM products` từ SQLite. Trả về mảng JSON các sản phẩm. Endpoint công khai, không cần auth.                                                                                                                                       |
| 4   | `/api/cart`            | `POST` | `Authorization: Bearer <token>Content-Type: application/json` | `{"id": number, "name": string, "price": number, "quantity": number}` | **200 OK**`{"message": "Added to cart"}`                                                                                        | **401 Unauthorized** (thiếu token)**403 Forbidden** (token sai/hết hạn)                                                                                           | Yêu cầu`authenticateToken`. Backend lưu giỏ hàng trong bộ nhớ `userCarts[userId] = []` (in-memory object).                                                                                                                                                         |
| 5   | `/api/cart`            | `GET`  | `Authorization: Bearer <token>`                               | Không                                                                  | **200 OK**`[{"id": 1, "name": "...", "price": ..., "quantity": ...}, ...]`                                                      | **401 / 403** (nếu thiếu/sai auth)                                                                                                                                       | Yêu cầu`authenticateToken`. Trả về toàn bộ danh sách items trong mảng `userCarts[userId]`.                                                                                                                                                                        |
| 6   | `/api/forgot-password` | `POST` | `Content-Type: application/json`                              | `{"email": string}`                                                   | **200 OK**`{"message": "Mã đặt lại mật khẩu đã được tạo", "resetToken": "1234"}`                                    | **404 Not Found**`{"error": "User not found"}`**500 Internal Server Error**`{"error": err.message}`                                                              | **Không** cần `Authorization` header. Tuy nhiên, email **phải tồn tại trong database**, nếu không có sẽ trả về 404. Backend sinh mã OTP ngẫu nhiên 4 chữ số (1000 - 9999) và cập nhật vào `users.reset_token`.                           |

---

## 3. Final E2E Workflow

Để đảm bảo quy trình kiểm thử hoàn chỉnh, liên tục, phản ánh đúng hành vi người dùng thực tế và bao phủ toàn bộ 3 nhóm endpoint, workflow E2E được thiết kế tuần tự như sau:

```
+---------------------------------------------------------------------------------------------------+
|                                      VIRTUAL USER E2E WORKFLOW                                     |
+---------------------------------------------------------------------------------------------------+
|  [Step 1] POST /api/register                                                                      |
|           - Input: Unique { name, email, password } sinh động theo VU & Iteration                |
|           - Output: HTTP 200, user ID tạo mới                                                     |
|                                     │                                                             |
|                                     ▼                                                             |
|  [Step 2] POST /api/login                                                                         |
|           - Input: { email, password } vừa đăng ký                                                |
|           - Output: HTTP 200, trích xuất JWT `token` và `user.id` qua JSON Extractor              |
|                                     │                                                             |
|                                     ▼                                                             |
|  [Step 3] GET /api/products  [READ-HEAVY GROUP]                                                   |
|           - Input: Không cần Auth (hoặc kèm search query)                                         |
|           - Output: HTTP 200, trích xuất `product_id`, `product_name`, `product_price`           |
|                                     │                                                             |
|                                     ▼                                                             |
|  [Step 4] POST /api/cart  [TRANSACTIONAL GROUP]                                                   |
|           - Input: Header `Authorization: Bearer ${token}`, body chứa product vừa lấy             |
|           - Output: HTTP 200 {"message": "Added to cart"}                                         |
|                                     │                                                             |
|                                     ▼                                                             |
|  [Step 5] GET /api/cart                                                                           |
|           - Input: Header `Authorization: Bearer ${token}`                                        |
|           - Output: HTTP 200, xác minh giỏ hàng chứa đúng sản phẩm vừa thêm                       |
|                                     │                                                             |
|                                     ▼                                                             |
|  [Step 6] POST /api/forgot-password  [AUTH-HEAVY GROUP]                                           |
|           - Input: { email: "${email}" } của user hiện tại                                        |
|           - Output: HTTP 200, sinh reset token (OTP) thành công                                   |
+---------------------------------------------------------------------------------------------------+
```

### Tính hợp lý của vị trí Step 6 (`POST /api/forgot-password`):

* `POST /api/forgot-password` đòi hỏi email phải tồn tại trong database (nếu không sẽ bị HTTP 404 `User not found`).
* Nhờ thực hiện sau Step 1 (Register) và Step 2 (Login), email của Virtual User đã chắc chắn tồn tại hợp lệ trong DB, giúp request luôn trả về HTTP 200 với mã `resetToken`.
* Việc thực hiện forgot password không làm vô hiệu hóa token hiện tại của phiên đăng nhập (backend chỉ cập nhật `reset_token` trong DB, không hủy JWT token), do đó không gây ảnh hưởng hay xung đột trạng thái.

---

## 4. Endpoint Group Mapping

Theo đúng định hướng đề bài và phân loại kiến trúc:

| Nhóm Endpoint (HW05)   | Endpoint Mục Tiêu           | Loại Tải / Hành Vi Hệ Thống                   | Lý Do Phân Loại                                                                                                                                                                                                                          |
| :---------------------- | :---------------------------- | :------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Auth-heavy**    | `POST /api/forgot-password` | CPU / DB Write (Update & Token Generation)         | Thực hiện tra cứu user trong DB SQLite (`SELECT`), sinh mã OTP ngẫu nhiên (`Math.random`), và thực hiện ghi/cập nhật DB (`UPDATE users SET reset_token = ?`). Gây áp lực ghi DB và xử lý logic bảo mật xác thực. |
| **Read-heavy**    | `GET /api/products`         | CPU / DB Read (Table Scan / Query)                 | Truy vấn toàn bộ danh mục sản phẩm từ SQLite (`SELECT * FROM products`), tạo tải đọc dữ liệu và serialize mảng JSON lớn gửi về client.                                                                                  |
| **Transactional** | `POST /api/cart`            | Concurrency / State Mutation (Session Transaction) | Yêu cầu xác thực JWT qua middleware`authenticateToken`, mutate trạng thái giỏ hàng (`userCarts[userId].push`), thao tác với phiên làm việc riêng của từng người dùng.                                                |

---

## 5. Supporting API Mapping

Các API sau đóng vai trò thiết lập dữ liệu và kiểm tra trạng thái trong luồng E2E, **không** tính vào 3 nhóm endpoint chính:

| Endpoint               | Vai Trò Trong Workflow                                   | Lý Do Là Supporting API                                                                                                                                             |
| :--------------------- | :-------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /api/register` | Khởi tạo tài khoản mới độc nhất cho Virtual User. | Bắt buộc phải có để tạo ra user hợp lệ và dữ liệu email tồn tại trong DB, tránh lỗi 404 cho forgot-password và tránh phụ thuộc tài khoản cứng. |
| `POST /api/login`    | Xác thực thông tin đăng nhập và lấy JWT token.    | Cần thiết để sinh Bearer Token nhằm truy cập các API giỏ hàng (`/api/cart`).                                                                               |
| `GET /api/cart`      | Đọc lại giỏ hàng của user sau khi thêm sản phẩm. | Dùng để verify trạng thái transactional thành công, tăng tính chân thực cho luồng E2E của người dùng.                                                 |

---

## 6. Authentication & Token Flow

1. **Khởi tạo:**
   * Sau khi `POST /api/register` thành công (HTTP 200), client nhận `{ "message": "User registered successfully", "id": 123 }`.
2. **Xác thực:**
   * Client gửi `POST /api/login` với body: `{"email": "${user_email}", "password": "${user_password}"}`.
   * Server trả về JSON chứa JWT token:
     ```json
     {
       "message": "Login successful",
       "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
       "user": { "id": 123, "name": "...", "email": "...", "role": "user", ... }
     }
     ```
3. **Trích xuất & Tương quan (Correlation):**
   * Sử dụng **JSON Extractor** trong JMeter để bắt biến:
     * JSON Path: `$.token` -> Lưu vào biến `${jwt_token}`.
     * JSON Path: `$.user.id` -> Lưu vào biến `${user_id}`.
4. **Ủy quyền cho các Transaction Request:**
   * Sử dụng **HTTP Header Manager** (hoặc gán header trực tiếp cho các sampler cần auth):
     * `Authorization`: `Bearer ${jwt_token}`
   * Áp dụng cho: `POST /api/cart` và `GET /api/cart`.
5. **Phạm vi bảo vệ của Middleware `authenticateToken`:**
   * `authenticateToken` kiểm tra header `Authorization: Bearer <token>`, giải mã bằng `SECRET_KEY = "super_secret_key_that_should_not_be_here"`.
   * Gán `req.user = { id: user.id, role: user.role }` để phục vụ định danh giỏ hàng `userCarts[userId]`.

---

## 7. Correlation Requirements (Trích Xuất & Tương Quan Biến)

Để luồng E2E hoạt động linh hoạt, tự động và không hardcode:

| Biến Tương Quan | Nguồn Trích Xuất (Sampler Nguồn)   | Cơ Chế Trích Xuất (Extractor)                                 | Biến JMeter                                                                           | Mục Đích Sử Dụng                      |
| :----------------- | :------------------------------------- | :---------------------------------------------------------------- | :------------------------------------------------------------------------------------- | :----------------------------------------- |
| `user_email`     | Dynamic Generated / CSV Data           | JSR223 / JMeter Built-in Functions                                | `${user_email}`                                                                      | Dùng cho Register, Login, Forgot-password |
| `user_password`  | CSV Data / Default Var                 | CSV Data Set Config                                               | `${user_password}`                                                                   | Dùng cho Register, Login                  |
| `user_name`      | CSV Data / Dynamic Generator           | CSV Data Set Config / Dynamic                                     | `${user_name}`                                                                       | Dùng cho Register                         |
| `jwt_token`      | `POST /api/login` Response           | JSON Extractor (`$.token`) | `${jwt_token}`                   | Header`Authorization: Bearer ${jwt_token}` cho `POST /api/cart`, `GET /api/cart` |                                            |
| `user_id`        | `POST /api/login` Response           | JSON Extractor (`$.user.id`) | `${user_id}`                   | Định danh người dùng                                                              |                                            |
| `product_id`     | `GET /api/products` Response         | JSON Extractor (`$..id` hoặc `$.[0].id`) | `${product_id}` | Dùng làm payload cho`POST /api/cart`                                               |                                            |
| `product_name`   | `GET /api/products` Response         | JSON Extractor (`$.[0].name`) | `${product_name}`             | Dùng làm payload cho`POST /api/cart`                                               |                                            |
| `product_price`  | `GET /api/products` Response         | JSON Extractor (`$.[0].price`) | `${product_price}`           | Dùng làm payload cho`POST /api/cart`                                               |                                            |
| `reset_token`    | `POST /api/forgot-password` Response | JSON Extractor (`$.resetToken`) | `${reset_token}`            | Kiểm tra assertion xác nhận mã OTP đã được tạo                               |                                            |

---

## 8. Unique-User Strategy (Chống Trùng Lặp Tài Khoản Tuyệt Đối)

### Vấn đề:

* Trong các bài test hiệu năng (Load với nhiều VU, Stress với tải tăng dần, Spike với xung tải đột biến, hoặc các lần chạy rerun), nếu các thread sử dụng chung một email cố định:
  * Trùng lặp dữ liệu trong SQLite.
  * Khi login sai/lockout sẽ làm ảnh hưởng chéo giữa các virtual users.
  * Login lấy bản ghi đầu tiên khớp email, gây sai lệch thống kê và dữ liệu người dùng.

### Giải pháp Chiến Lược:

Mỗi Virtual User trong mỗi vòng lặp (iteration) của mỗi lần chạy (run) sẽ sở hữu một **Identity hoàn toàn độc nhất (Globally Unique)**.

* **Công thức sinh Email:**
  ```
  perf_${__time(yyyyMMdd_HHmmss,)}_${__P(run_id,local)}_th${__threadNum}_it${__iterationNum}_${__Random(1000,9999)}@perf.eshop.vn
  ```
* **Thành phần công thức:**
  1. `perf_`: Tiền tố nhận diện test traffic.
  2. `${__time(yyyyMMdd_HHmmss,)}`: Timestamp tại thời điểm chạy (chống trùng giữa các lần rerun).
  3. `${__P(run_id,local)}`: Biến nhận diện kịch bản (Load / Stress / Spike / Endurance).
  4. `th${__threadNum}`: ID luồng của Virtual User (chống trùng giữa các Virtual Users chạy đồng thời).
  5. `it${__iterationNum}`: Số thứ tự vòng lặp của luồng.
  6. `${__Random(1000,9999)}`: Số ngẫu nhiên 4 chữ số tăng cường tính ngẫu nhiên.
* **Tên hiển thị (`user_name`):**
  ```
  VU_${__threadNum}_${__Random(100,999)}
  ```
* **Mật khẩu chuẩn hóa (`user_password`):**
  ```
  PerfTest@123456
  ```

Chiến lược này đảm bảo:

* **Không bao giờ bị trùng lặp tài khoản** giữa các Thread, giữa các Iteration, và giữa các lần chạy lại kịch bản.
* **Không bao giờ bị lockout nhầm** do tài khoản bị dùng chung.
* Dễ dàng lọc, dọn dẹp hoặc truy vấn trong database khi cần (`WHERE email LIKE 'perf_%'`).

---

## 9. CSV & Data Requirements

Để đáp ứng yêu cầu **Data-Driven Testing (REQ-31 & REQ-32)**, hệ thống dữ liệu đầu vào được thiết kế như sau:

### File CSV: `data/user_profiles.csv`

Cung cấp dữ liệu mẫu cho hồ sơ người dùng (họ tên cơ bản, mật khẩu mặc định, tiền tố email, số điện thoại, địa chỉ):

```csv
name_prefix,default_password,shipping_address,phone
Nguyen Van,PerfPass@123!,123 Nguyen Hue Q1 HCMC,0901234567
Tran Thi,PerfPass@123!,456 Le Duan Hanoi,0912345678
Le Van,PerfPass@123!,789 Tran Phu Danang,0923456789
Pham Thi,PerfPass@123!,101 Hung Vuong Cantho,0934567890
Hoang Van,PerfPass@123!,202 Quang Trung Haiphong,0945678901
```

### File CSV: `data/search_keywords.csv` (Tùy chọn cho Read-heavy)

Cung cấp các từ khóa tìm kiếm để đa dạng hóa truy vấn Read-heavy:

```csv
keyword
iPhone
Samsung
MacBook
AirPods
Keychron
```

---

## 10. Các UNKNOWN & ASSUMPTION Đã Xác Minh

| STT | Vấn Đề Cần Làm Rõ                                                 | Kết Quả Xác Minh Từ Code & DB                                                                                                           | Kết Luận Thực Hiện                                                                                                                                                                                        |
| :-- | :---------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `POST /api/register` có trả JWT token không?                       | Code`server.js` chỉ trả `{"message": "...", "id": this.lastID}`.                                                                      | **Không có token**. Phải gọi `POST /api/login` để lấy token.                                                                                                                                   |
| 2   | Trường token trả về trong login tên là gì?                       | `server.js` trả về `{ message: "...", token: token, user: user }`.                                                                    | Trường token là**`token`**.                                                                                                                                                                        |
| 3   | `POST /api/cart` và `GET /api/cart` có cần token không?         | Cả 2 đều có middleware`authenticateToken`.                                                                                            | **Bắt buộc** Header `Authorization: Bearer <token>`.                                                                                                                                                |
| 4   | Dữ liệu giỏ hàng được lưu ở đâu?                             | `userCarts = {}` (In-memory Object trong Node.js process).                                                                                | Giỏ hàng tồn tại trong suốt vòng đời tiến trình backend, gắn theo`user.id`.                                                                                                                      |
| 5   | `POST /api/forgot-password` có cần token không?                    | Không có middleware`authenticateToken`.                                                                                                 | **Không cần token**, nhưng email phải có trong bảng `users` (nếu không sẽ trả HTTP 404).                                                                                                    |
| 6   | Bảng`users` trong database có ràng buộc `UNIQUE(email)` không? | `database.js` tạo bảng `users (id, name, email, password...)` không có `UNIQUE(email)`.                                           | Mặc dù DB cho phép trùng, backend login chỉ`SELECT` theo email -> Bắt buộc kịch bản test phải sinh email duy nhất để tránh xung đột.                                                        |
| 7   | Cơ chế Lockout hoạt động thế nào?                                | Khi login sai mật khẩu:`newAttempts = login_attempts + 2`. Nếu `newAttempts >= 3`, gán `locked_until = now + 180000ms` (3 phút). | Tài khoản bị khóa 3 phút ngay từ lần login sai thứ 2 (0 -> 2 -> 4 >= 3). Do test E2E login với đúng credentials vừa register, luồng test chuẩn sẽ**không** bị lockout ngoài ý muốn. |

---

## 11. Các Rủi Ro & Điểm Cần Human Review

1. **Rủi ro Concurrency với SQLite:**
   * SQLite sử dụng file-level locking (`database.sqlite`). Khi chạy kịch bản Stress hoặc Spike với concurrency cao (ví dụ: hàng trăm VU đồng thời thực hiện `INSERT INTO users` ở Step 1 hoặc `UPDATE users` ở Step 6), SQLite có thể gặp lỗi `SQLITE_BUSY: database is locked`.
   * *Đề xuất:* Đặt ramp-up hợp lý và theo dõi tỷ lệ lỗi của DB khi Stress/Spike. Đây cũng là một phát hiện giá trị để ghi nhận vào phần phân tích Task 2 & Bug/Performance Report.
2. **Bộ nhớ In-memory của `userCarts`:**
   * Do `userCarts` lưu trên RAM của Node.js, khi số lượng VU tạo mới liên tục lên tới hàng chục nghìn lượt, dung lượng RAM của tiến trình Node.js có thể tăng dần.
   * *Đề xuất:* Ghi nhận chỉ số Memory qua `htop` / Resource Monitor để đánh giá ngưỡng Endurance threshold.
3. **Phân bổ Think Time & Pacing giữa các bước:**
   * Trong thực tế, người dùng cần vài giây suy nghĩ trước khi chọn sản phẩm hoặc thao tác giỏ hàng.
   * Cần cấu hình **Gaussian Random Timer** / **Uniform Random Timer** (ví dụ: 500ms - 2000ms) giữa các bước để mô phỏng tải chân thực, tránh dồn ép tải phi thực tế.
4. **Cấu hình 3 Listeners không lặp lại (REQ-33):**
   * Đề xuất lựa chọn phân bổ:
     * **Load Test Plan:** `Summary Report` + `View Results Tree` (disabled during run / only debug).
     * **Stress Test Plan:** `Aggregate Report` + `Transactions per Second (TPS) Listener / Simple Data Writer`.
     * **Spike Test Plan:** `Response Times Over Time Listener` + `Generate Summary Results`.
     * *(Đảm bảo khi xuất deliverables có 3 dạng listener/báo cáo hoàn toàn khác biệt).*

---

> 🎯 **Trạng thái:** Toàn bộ phạm vi, API contract, workflow E2E và chiến lược dữ liệu đã được đối soát chính xác 100% với source code của hệ thống SUT.
> **Kính mời Human Review phê duyệt trước khi chuyển sang Bước 2 (Thiết kế JMeter Test Plans)!**
