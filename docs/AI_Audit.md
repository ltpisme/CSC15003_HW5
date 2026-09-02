# AI Audit - HW05

## 1. Thiết kế workflow

<a id="log-1"></a>

- **Timestamp**: 23:49 01/09/2005
- **Tools**: ChatGPT
- **Input**: `extracted_requirement.md`
- **Prompt**:

==Prompt Start==

[Nhiệm vụ]
Với yêu cầu trong HW5, hãy thiết kế cho tôi workflow e2e cho 3 api POST forgot password, GET product, POST cart
[Yêu cầu]
được đính kèm

==Promp End==

- **Output**

==Output Start==

Dựa trên HW05, workflow E2E của bạn cần **đi qua cả 3 nhóm endpoint**: Auth-heavy → Read-heavy → Transactional, và cả **Load / Stress / Spike phải dùng cùng một workflow**.

Với 3 API bạn chọn, tôi đề xuất thiết kế như sau:

## 1. E2E workflow

```text
CSV Test Data
     │
     ▼
POST /forgot-password
     │
     │ Auth-heavy
     ▼
GET /products
     │
     │ Read-heavy
     ▼
POST /cart
     │
     │ Transactional
     ▼
End of workflow
```

Tuy nhiên, có **một điểm cần lưu ý rất quan trọng**: HW05 đưa ví dụ Auth-heavy là login/lockout, Read-heavy là product listing/search/detail và Transactional là add-to-cart/checkout.

Vì vậy, nếu API `POST forgot-password` thực sự thuộc nhóm **Auth-heavy** trong SUT của bạn, thì lựa chọn này có thể dùng được. Nhưng cần xác minh API thực tế của EShop trước khi chốt, thay vì tự gán nhóm endpoint.

---

# 2. Chi tiết workflow đề xuất

### Step 1 — `POST /forgot-password`

**Mục đích:** tạo/kiểm tra password-reset request.

Ví dụ:

```http
POST /forgot-password
Content-Type: application/json

{
  "email": "${email}"
}
```

**CSV:**

```csv
email
user01@example.com
user02@example.com
user03@example.com
```

JMeter sẽ đọc CSV bằng **CSV Data Set Config**, sau đó `${email}` được inject vào request.

**Validation nên có:**

* HTTP status đúng theo API
* Response không phải server error
* Response body có cấu trúc mong đợi
* Không phụ thuộc vào giá trị cố định do AI tự đoán

---

### Step 2 — `GET /products`

Sau khi bước forgot-password hoàn thành:

```http
GET /products
```

Nếu API hỗ trợ query:

```http
GET /products?category=${category}
```

hoặc:

```http
GET /products?search=${keyword}
```

CSV có thể mở rộng:

```csv
email,category,keyword,productId
user01@example.com,electronics,phone,101
user02@example.com,books,testing,102
user03@example.com,clothing,shirt,103
```

**Validation:**

* HTTP status thành công
* Response có danh sách product
* Có thể kiểm tra JSON structure
* Nếu workflow cần product cụ thể cho bước sau, **extract `productId` từ response**, thay vì hard-code.

Ví dụ:

```text
GET /products
       │
       ▼
JSON Extractor
       │
       └── productId
                │
                ▼
           POST /cart
```

Đây là điểm làm workflow thực sự **E2E/data-dependent**, thay vì chỉ chạy 3 request độc lập.

---

### Step 3 — `POST /cart`

Dùng dữ liệu hoặc `productId` lấy từ bước GET:

```http
POST /cart
Content-Type: application/json

{
  "productId": "${productId}",
  "quantity": "${quantity}"
}
```

CSV:

```csv
email,category,keyword,quantity
user01@example.com,electronics,phone,1
user02@example.com,books,testing,2
user03@example.com,clothing,shirt,1
```

**Validation:**

* HTTP status đúng
* Cart operation thành công
* Response chứa thông tin cart/item mong đợi

---

# 3. Nhưng có một vấn đề về E2E

Nếu `forgot-password` **không tạo session/authentication state**, còn `GET /products` và `POST /cart` không phụ thuộc vào nó, thì:

```text
forgot-password
      ↓
GET products
      ↓
POST cart
```

về mặt kỹ thuật chỉ là **chuỗi 3 API**, chưa chắc là một workflow E2E có dependency thực sự.

Tôi khuyến nghị kiểm tra dependency như sau:

| Step | API                      | Nhóm         | Output dùng cho step sau? |
| ---- | ------------------------ | ------------- | -------------------------- |
| 1    | POST`/forgot-password` | Auth-heavy    | Có/không                 |
| 2    | GET`/products`         | Read-heavy    | `productId` → Step 3    |
| 3    | POST`/cart`            | Transactional | Kết thúc                 |

Ít nhất nên có:

```text
GET /products
      ↓
extract productId
      ↓
POST /cart(productId)
```

Còn quan hệ giữa `forgot-password` → `GET /products` cần xác minh từ API thực tế.

---

# 4. Cấu trúc JMeter nên làm

Tôi sẽ thiết kế **một workflow chung**, sau đó tạo 3 test plan dựa trên workflow này:

```text
Test Plan
│
├── User Defined Variables
│
├── CSV Data Set Config
│
├── HTTP Request Defaults
│
├── HTTP Header Manager
│
└── Thread Group
    │
    ├── POST /forgot-password
    │   ├── Assertion
    │   └── Extractor (nếu cần)
    │
    ├── GET /products
    │   ├── Assertion
    │   └── JSON Extractor
    │
    ├── POST /cart
    │   └── Assertion
    │
    └── [Listener]
```

Sau đó:

```text
LOAD
 └── E2E workflow trên

STRESS
 └── E2E workflow trên

SPIKE
 └── E2E workflow trên
```

Điều này phù hợp trực tiếp với yêu cầu HW05 rằng **3 scenario phải chạy trên cùng một E2E workflow**.

---

# 5. Data-driven design

HW05 bắt buộc workflow phải được tham số hóa bằng CSV.

Tôi đề xuất một file:

```text
test-data.csv
```

Ví dụ:

```csv
email,keyword,quantity
user01@example.com,phone,1
user02@example.com,laptop,2
user03@example.com,book,1
```

JMeter:

```text
CSV Data Set Config
        │
        ├── ${email}
        ├── ${keyword}
        └── ${quantity}
```

Mỗi Virtual User lấy một dòng dữ liệu.

**Không nên** để AI tự tạo dữ liệu giả rồi coi đó là dữ liệu thật. Bạn phải dùng dữ liệu phù hợp với SUT thực tế và có thể giải thích nguồn dữ liệu trong báo cáo.

---

# 6. 3 scenario nhưng chỉ 1 workflow

Đây là phần rất quan trọng của HW05.

### Load

```text
Low → Normal expected load
       │
       ▼
POST forgot-password
       ↓
GET products
       ↓
POST cart
```

### Stress

```text
Normal → Increasing load → Beyond expected load
       │
       ▼
POST forgot-password
       ↓
GET products
       ↓
POST cart
```

### Spike

```text
Normal load
    │
    │ sudden increase
    ▼
HIGH CONCURRENCY
    │
    ├── POST forgot-password
    ├── GET products
    └── POST cart
```

Khác nhau **ở workload model**, không phải ở API workflow.

HW05 cũng yêu cầu AI hỗ trợ lựa chọn `think-time`, `ramp-up`, số thread/virtual users cho từng scenario và sau đó bạn phải review lại các lựa chọn đó.

---

# 7. Listener nên phân biệt

HW05 yêu cầu **3 loại listener/report khác nhau**, không lặp lại giữa 3 scenario.

Có thể bố trí:

```text
Load
└── Aggregate Report

Stress
└── Summary Report

Spike
└── View Results Tree
```

Nhưng khi chạy performance test thực tế, **View Results Tree không nên bật với tải lớn**, vì chính JMeter GUI/listener có thể làm ảnh hưởng đến kết quả.

Do đó tôi khuyên dùng các report phù hợp hơn cho từng scenario và chỉ dùng GUI listener khi cần minh chứng/debug.

---

# 8. Workflow cuối cùng tôi khuyên bạn chốt

```text
                    CSV
                     │
                     ▼
              ┌─────────────┐
              │ Virtual User│
              └──────┬──────┘
                     │
                     ▼
        ┌────────────────────────┐
        │ POST /forgot-password  │
        │     AUTH-HEAVY         │
        └───────────┬────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │      GET /products     │
        │      READ-HEAVY        │
        └───────────┬────────────┘
                    │
                    │ extract productId
                    ▼
        ┌────────────────────────┐
        │       POST /cart       │
        │      TRANSACTIONAL     │
        └───────────┬────────────┘
                    │
                    ▼
                   END
```

Sau đó dùng **chính workflow này** cho:

```text
              ┌───────────────┐
              │  E2E Workflow │
              └───────┬───────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        LOAD        STRESS       SPIKE
```

Điểm tôi **chưa muốn chốt ngay** là `POST /forgot-password` có thực sự là Auth-heavy và ba API này có tạo thành dependency E2E hợp lệ hay không. File HW05 chỉ quy định **3 nhóm endpoint và ví dụ**, chứ không cung cấp contract cụ thể của ba API này.

Nếu bạn gửi **README/spec hoặc source code của EShop SUT**, tôi có thể dựa trực tiếp vào API contract để thiết kế chính xác **request body → authentication → correlation/extractor → CSV → assertions → JMeter tree** cho 3 API này, tránh thiết kế sai endpoint.

==Output End==

- **Verdict**: Incomplete
- **Reasoning**: Thiết kế workflow tổng quát hợp lí, cần đối chiếu thêm với `api_specification.md`
- **Student fix**: Không cần thiết


## 2. Đối chiếu thiết kế workflow

<a id="log-2"></a>

- **Timestamp**: 23:58 01/09/2005
- **Tools**: ChatGPT
- **Input**: `api_specification`, workflow được thiết kế ở phần 1
- **Prompt**:

==Prompt Start==
[Nhiệm vụ]
Đối chiếu lại với tài liệu đặc tả api, thiết kế lại workflow, bắt buộc phải sử dụng api đã chọn: POST forgot password, GET products, POST cart. Có thể sử dụng API khác hỗ trợ cho hợp lí với luồng E2E nhưng chỉ đánh giá Performance Metric cho API được chọn. Thiết kế Workflow, test data cần thiết, performance metric cho phù hợp yêu cầu HW05

[Đặc tả API]
được đính kèm
==Promp End==

- **Output**

==Output Start==
Có. Với API specification bạn cung cấp, workflow trước cần **thiết kế lại** vì `POST /api/forgot-password` không tạo authentication session cho các API phía sau. Trong khi `POST /api/cart` bắt buộc JWT.

Tôi đề xuất giữ **bắt buộc 3 API bạn đã chọn** nhưng thêm API hỗ trợ để tạo một E2E dependency hợp lý.

## 1. Workflow E2E nên chốt

```text
POST /api/login
      │
      │  JWT token
      ▼
POST /api/forgot-password
      │
      │  Auth-heavy target
      ▼
GET /api/products
      │
      │  extract product id
      ▼
POST /api/cart
      │
      │  Transactional target
      ▼
GET /api/cart
      │
      ▼
End
```

### Phân loại

| Step | Endpoint                    | Vai trò                      | Đánh giá performance? |
| ---- | --------------------------- | ---------------------------- | --------------------- |
| 1    | `POST /api/login`           | Supporting API – lấy JWT     | **Không**             |
| 2    | `POST /api/forgot-password` | **Auth-heavy – selected**    | **Có**                |
| 3    | `GET /api/products`         | **Read-heavy – selected**    | **Có**                |
| 4    | `POST /api/cart`            | **Transactional – selected** | **Có**                |
| 5    | `GET /api/cart`             | Supporting API – verify cart | **Không**             |

Cách này đáp ứng yêu cầu quan trọng của HW05: **một E2E workflow duy nhất đi qua cả 3 nhóm endpoint**, đồng thời vẫn chỉ đánh giá performance metrics của 3 API bạn đã chọn. HW05 yêu cầu 3 scenario Load/Stress/Spike phải dùng cùng workflow E2E. 

---

# 2. Tại sao phải thêm `POST /login`?

Theo API specification:

```text
POST /api/login
       ↓
JWT token
       ↓
Authorization: Bearer <token>
       ↓
POST /api/cart
```

`POST /api/forgot-password` chỉ nhận:

```json
{
  "email": "test@domain.com"
}
```

và trả:

```json
{
  "message": "Mã đặt lại mật khẩu đã được tạo",
  "resetToken": "123456"
}
```

Nó **không trả JWT**.

Trong khi `/api/cart` yêu cầu:

```http
Authorization: Bearer <token>
```

Do đó không thể thiết kế hợp lý:

```text
forgot-password
      ↓
products
      ↓
cart
```

nếu không có login ở đâu đó.

`POST /api/login` là supporting API hợp lý nhất để thiết lập authentication context.

---

# 3. Vai trò của `forgot-password`

Có một điểm đặc biệt:

```text
POST /api/login
       ↓
POST /api/forgot-password
```

Về business flow, login trước forgot-password không phải cách user thông thường sử dụng chức năng quên mật khẩu.

Vì vậy tôi **không nên mô tả workflow này là "user forgot password rồi tiếp tục mua hàng"**.

Thay vào đó, trong báo cáo nên mô tả:

> E2E performance workflow được thiết kế để exercise ba API thuộc ba workload categories trong cùng một authenticated virtual-user session. Login là supporting step để establish JWT authentication; forgot-password, product listing và add-to-cart là ba selected endpoints được đo performance. Forgot-password không cung cấp JWT và không tạo dependency authentication cho các bước sau.

Điều này trung thực hơn với API specification.

---

# 4. Workflow chi tiết trong JMeter

## Step 0 — CSV Data

Tôi đề xuất **không đưa password-reset token vào CSV**.

Token phải được tạo dynamically từ `forgot-password` response nếu cần.

CSV:

```csv
email,password,search,quantity
perf01@example.com,Password123!,phone,1
perf02@example.com,Password123!,laptop,2
perf03@example.com,Password123!,book,1
perf04@example.com,Password123!,shirt,1
```

Có thể dùng:

```text
${email}
${password}
${search}
${quantity}
```

HW05 yêu cầu data-driven workflow bằng CSV để parameterize requests. 

---

# 5. Step 1 — Login

### Request

```http
POST /api/login
Content-Type: application/json
```

```json
{
  "email": "${email}",
  "password": "${password}"
}
```

### Response

API specification nói login trả:

```text
token + user
```

Dùng JSON Extractor:

```text
JSON Extractor
Variable name: authToken
JSON Path: <path thực tế của token>
```

Sau đó:

```http
Authorization: Bearer ${authToken}
```

**Quan trọng:** JSON path phải kiểm tra bằng response thực tế. Không nên tự giả định token nằm ở `$.token` nếu chưa chạy API.

### Performance metric

**Không đánh giá login.**

Nó chỉ là setup/supporting request.

---

# 6. Step 2 — Forgot Password

Đây là **Selected API #1**.

```http
POST /api/forgot-password
Content-Type: application/json
```

```json
{
  "email": "${email}"
}
```

Expected:

```http
200 OK
```

Response:

```json
{
  "message": "Mã đặt lại mật khẩu đã được tạo",
  "resetToken": "123456"
}
```

Có thể thêm JSON Extractor:

```text
resetToken
```

nhưng **không cần dùng token này cho workflow**, vì `POST /api/reset-password` không phải API được chọn.

Nếu muốn chứng minh E2E data flow mạnh hơn, có thể gọi:

```text
POST /api/reset-password
```

nhưng tôi **không khuyến nghị đưa nó vào baseline workflow**, vì reset password sẽ làm phức tạp dữ liệu account giữa các iteration và có thể phá login cho những vòng sau.

---

# 7. Step 3 — Get Products

Đây là **Selected API #2**.

```http
GET /api/products?search=${search}
```

Ví dụ:

```http
GET /api/products?search=phone
```

Response phải được dùng để tìm product.

### JSON Extractor

Extract:

```text
productId
productName
productPrice
```

**Nhưng JSONPath phải dựa trên response thật.**

Ví dụ nếu response thực tế là:

```json
[
  {
    "id": 1,
    "name": "Phone",
    "price": 100000
  }
]
```

thì mới dùng:

```text
$.0.id
```

hoặc JSONPath tương ứng theo format thực tế.

Không nên hard-code `productId=1`.

---

# 8. Step 4 — Add to Cart

Đây là **Selected API #3**.

Endpoint:

```http
POST /api/cart
Authorization: Bearer ${authToken}
Content-Type: application/json
```

Body:

```json
{
  "id": ${productId},
  "name": "${productName}",
  "price": ${productPrice},
  "quantity": ${quantity}
}
```

Đây là điểm rất quan trọng.

Theo API specification, `/api/cart` **không chỉ nhận product ID** mà nhận:

```json
{
  "id": 1,
  "name": "Sản phẩm A",
  "price": 100000,
  "quantity": 2
}
```

Do đó workflow nên lấy:

```text
GET /products
      │
      ├── productId
      ├── productName
      └── productPrice
              │
              ▼
        POST /cart
```

Như vậy mới có **data dependency thật sự** giữa Read-heavy → Transactional.

---

# 9. Step 5 — Get Cart

Có thể thêm:

```http
GET /api/cart
Authorization: Bearer ${authToken}
```

Mục đích:

```text
POST /cart
      ↓
GET /cart
      ↓
verify item exists
```

Đây là **supporting API**, không tính performance metric.

Tuy nhiên có một trade-off:

### Nếu mục tiêu là performance test đơn giản

Dùng:

```text
Login
 ↓
Forgot password
 ↓
Get products
 ↓
Add cart
```

### Nếu mục tiêu là E2E evidence mạnh

Dùng:

```text
Login
 ↓
Forgot password
 ↓
Get products
 ↓
Add cart
 ↓
Get cart
```

Tôi nghiêng về **workflow thứ hai** vì nó thể hiện transaction đã hoàn thành và có verification.

---

# 10. Performance Metrics

HW05 yêu cầu thu thập performance metrics và resource monitoring. 

Bạn nên **tách rõ metric của selected APIs và supporting APIs**.

### Metrics chính

| Metric                   | Forgot Password | GET Products | POST Cart |
| ------------------------ | --------------: | -----------: | --------: |
| Request count            |               ✓ |            ✓ |         ✓ |
| Throughput / RPS         |               ✓ |            ✓ |         ✓ |
| Average response time    |               ✓ |            ✓ |         ✓ |
| Median / P50             |               ✓ |            ✓ |         ✓ |
| P90                      |               ✓ |            ✓ |         ✓ |
| **P95**                  |               ✓ |            ✓ |         ✓ |
| P99                      |               ✓ |            ✓ |         ✓ |
| Min                      |               ✓ |            ✓ |         ✓ |
| Max                      |               ✓ |            ✓ |         ✓ |
| Error rate               |               ✓ |            ✓ |         ✓ |
| HTTP status distribution |               ✓ |            ✓ |         ✓ |

Tôi đặc biệt khuyên lấy:

**P95 + throughput + error rate** làm các metric chính.

P95 cũng phù hợp với yêu cầu Task 3 của HW05 về theo dõi degradation của p95. 

---

# 11. Resource metrics

Ngoài JMeter metrics, bạn cần monitor backend resource usage vì HW05 yêu cầu screenshot tool + resource monitor đồng thời. 

Theo dõi tối thiểu:

| Resource       | Metric                    |
| -------------- | ------------------------- |
| CPU            | % usage                   |
| RAM            | Used / Available          |
| Process memory | Backend process           |
| Load           | System load nếu dùng htop |
| Network        | Có thể ghi nhận nếu cần   |

Ví dụ với Linux:

```text
JMeter
┌──────────────────────┐
│ Running E2E test     │
│ Threads: 100         │
│ Requests...          │
└──────────────────────┘

htop
┌──────────────────────┐
│ CPU: 78%             │
│ MEM: 61%             │
│ node: 1.2 GB         │
└──────────────────────┘
```

---

# 12. Đo metric thế nào cho đúng?

Một điểm tôi khuyên bạn làm khác với cách thiết kế đơn giản ban đầu:

**Không chỉ lấy metric của toàn bộ workflow.**

JMeter nên có Transaction Controller:

```text
E2E Workflow
│
├── Login                  [supporting]
│
├── Transaction: ForgotPassword
│
├── Transaction: GetProducts
│
├── Transaction: AddToCart
│
└── GetCart                [supporting]
```

Như vậy report có thể phân biệt:

```text
ForgotPassword
GetProducts
AddToCart
```

và bạn có thể báo cáo chính xác performance của **3 selected APIs**.

Nếu chỉ nhìn:

```text
E2E Workflow = 850 ms
```

thì không biết API nào gây bottleneck.

---

# 13. Test data nên thiết kế như thế nào?

Tôi đề xuất:

```text
data/
└── users.csv
```

```csv
email,password,search,quantity
perf01@example.com,Password123!,phone,1
perf02@example.com,Password123!,laptop,1
perf03@example.com,Password123!,book,2
perf04@example.com,Password123!,shirt,1
perf05@example.com,Password123!,camera,1
```

### Data flow

```text
email/password
      │
      ▼
    LOGIN
      │
      └── authToken
              │
              ├──────────────┐
              ▼              ▼
       forgot-password   POST cart
              │              ▲
              │              │
              ▼              │
        GET products ────────┘
          │
          ├── productId
          ├── productName
          └── productPrice
```

---

# 14. Một vấn đề quan trọng với performance test

`POST /api/forgot-password` có khả năng tạo **reset token/state mỗi lần gọi**.

Vì vậy **không nên dùng một email duy nhất cho hàng nghìn requests** nếu backend có rate-limit, state management hoặc DB side effect.

Nên có đủ test accounts:

```text
perf01
perf02
perf03
...
perf100
```

và CSV:

```csv
email,password,search,quantity
...
```

JMeter CSV Data Set Config có thể:

```text
Recycle on EOF: true
Stop thread on EOF: false
Sharing mode: All threads
```

hoặc cấu hình khác tùy mục tiêu.

**Nhưng số lượng account cần bao nhiêu phải xác định sau khi kiểm tra behavior thực tế của SUT**, không nên tùy tiện chọn 100/1000.

---

# 15. Load / Stress / Spike

Điểm quan trọng: **không thay đổi workflow**.

### Load

```text
Login
 ↓
Forgot Password
 ↓
Get Products
 ↓
Add Cart
 ↓
Get Cart
```

Chỉ thay đổi workload:

```text
Gradual ramp-up
Normal expected users
```

### Stress

```text
Same workflow
```

nhưng:

```text
Normal
 ↓
Increase users
 ↓
Continue until degradation
```

Theo HW05, Stress phải giúp tìm giới hạn hệ thống, và nếu kích hoạt lockout thì phải reset state giữa các lần chạy. 

### Spike

```text
Low/normal users
       │
       │ sudden increase
       ▼
Large concurrent users
       │
       ▼
Same E2E workflow
```

---

# 16. Endurance test

Ngoài 3 scenario chính, HW05 yêu cầu xác định endurance threshold bằng test duy trì khoảng **10–15 phút**. 

Có thể dùng cùng workflow:

```text
Stable load
     │
     │ 10–15 min
     ▼
Login
 ↓
Forgot Password
 ↓
Get Products
 ↓
Add Cart
 ↓
Get Cart
```

Theo dõi:

```text
p95
RPS
error rate
CPU
RAM
backend process memory
```

Sau đó kết luận dạng:

```text
Endurance threshold:
X virtual users

Stable throughput:
Y RPS

P95:
Z ms

Error rate:
N%

Backend memory:
M GB → M2 GB
```

**Các con số này phải lấy từ kết quả chạy thật**, không nên đặt trước.

---

# 17. Cấu trúc JMeter cuối cùng

Tôi đề xuất skeleton như sau:

```text
Test Plan
│
├── User Defined Variables
│
├── CSV Data Set Config
│
├── HTTP Request Defaults
│
├── HTTP Header Manager
│
└── Thread Group
    │
    └── E2E Workflow
        │
        ├── POST /api/login
        │   └── JSON Extractor → authToken
        │
        ├── Transaction Controller
        │   └── POST /api/forgot-password
        │
        ├── Transaction Controller
        │   └── GET /api/products
        │       └── JSON Extractor
        │           ├── productId
        │           ├── productName
        │           └── productPrice
        │
        ├── Transaction Controller
        │   └── POST /api/cart
        │
        └── GET /api/cart
```

Trong report, chỉ lấy:

```text
TARGET PERFORMANCE METRICS
├── POST /api/forgot-password
├── GET /api/products
└── POST /api/cart
```

Còn:

```text
SUPPORTING REQUESTS
├── POST /api/login
└── GET /api/cart
```

được dùng để hoàn thiện E2E nhưng **không dùng làm endpoint chính để đánh giá performance**.

---

## 18. Kết luận thiết kế

Tôi sẽ **chốt workflow này** cho HW05:

```text
                  CSV
                   │
                   ▼
             POST /login
             [SUPPORTING]
                   │
                   │ JWT
                   ▼
        POST /forgot-password
          ★ AUTH-HEAVY ★
                   │
                   ▼
          GET /api/products
           ★ READ-HEAVY ★
                   │
             extract product
                   │
                   ▼
            POST /api/cart
          ★ TRANSACTIONAL ★
                   │
                   ▼
             GET /api/cart
              [VERIFY]
                   │
                   ▼
                  END
```

**3 API được chấm performance:**

1. `POST /api/forgot-password`
2. `GET /api/products`
3. `POST /api/cart`

**API hỗ trợ:**

1. `POST /api/login` — lấy JWT.
2. `GET /api/cart` — verify transaction.

**Metric chính:**

* Throughput/RPS
* Average
* P50
* P90
* **P95**
* P99
* Max
* Error rate
* HTTP status
* CPU/RAM/backend process resource usage

Và quan trọng nhất, **cả Load, Stress, Spike và Endurance đều chạy cùng workflow trên**, chỉ thay đổi workload model. Điều này bám sát yêu cầu HW05 về một E2E workflow chung cho cả ba scenario. 
==Output End==

- **Verdict**: Valid
- **Reasoning**: Thiết kế workflow hợp lí, sử dụng kết hợp với API hỗ trợ cho end-to-end flow, test data và metric được gợi ý hợp lí, phù hợp với yêu cầu HW05
- **Student fix**: Không cần thiết
