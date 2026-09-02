# HW05 — Step 3: JMeter Test Plans Validation Report

> **Tài liệu:** `temp/AI/03_validation.md`  
> **Các tệp kịch bản được kiểm tra:**
> * `jmeter/23127452_Load_20260902.jmx`
> * `jmeter/23127452_Stress_20260902.jmx`
> * `jmeter/23127452_Spike_20260902.jmx`
> * `jmeter/data/user_profiles.csv`
> 
> **Trạng thái tổng thể sau khi sửa:** **PASS (Ready for Step 4 Human Review & Official Run)**

---

## 1. Kết Quả Kiểm Tra Đa Mức (Multi-Level Validation Matrix)

| Hạng Mục Kiểm Tra | Kịch Bản Load | Kịch Bản Stress | Kịch Bản Spike | Đánh Giá (Status) | Ghi Chú Kỹ Thuật |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **A. Cú pháp XML & Cấu trúc (Structural Validation)** | PASS | PASS | PASS | **PASS** | Cả 3 file JMX được parse và validate thành công, cây DOM chuẩn xác, không còn lỗi chèn text hay sai tag. |
| **B. Tương thích Apache JMeter 5.6.3 (GUI/Engine Load)** | PASS | PASS | PASS | **PASS** | JMeter 5.6.3 nạp Test Plan thành công 100% không phát sinh `NumberFormatException` hay `ConversionException` của XStream. |
| **C. Đường dẫn CSV (Relative Paths)** | `data/user_profiles.csv` | `data/user_profiles.csv` | `data/user_profiles.csv` | **PASS** | Sử dụng relative path `data/user_profiles.csv` từ thư mục chứa JMX; đã xóa `search_keywords.csv` không sử dụng. |
| **D. Common E2E Workflow** | 6 Samplers | 6 Samplers | 6 Samplers | **PASS** | Thứ tự 6 bước đồng nhất: `POST /register` -> `POST /login` -> `GET /products` -> `POST /cart` -> `GET /cart` -> `POST /forgot-password`. |
| **E. Phân Bổ Listener (REQ-33)** | **View Results Tree** | **Aggregate Report** | **Summary Report** | **PASS** | 3 listener khác nhau (`ViewResultsFullVisualizer`, `StatVisualizer`, `SummaryReport`) với cấu hình `SampleSaveConfiguration` hợp lệ (`assertionsResultsToSave = 0`). |
| **F. Workload Profile Topology** | 1 Thread Group (20 VU, 30s ramp, 225s dur) | 4 Thread Groups (Staircase 25 → 50 → 75 → 100 VU) | 2 Thread Groups (5 VU baseline → 80 VU surge → 5 VU recovery) | **PASS** | Khởi tạo topology tải chính xác theo thiết kế bằng native Thread Groups mà không cần plugin bên ngoài. |
| **G. Timers & Pacing** | Gaussian (500ms ± 200ms) | Uniform (100–300ms) | Gaussian (100ms ± 50ms) | **PASS** | Đúng thiết kế pacing cho từng loại tải. |
| **H. Correlation & Extractions** | 4 biến nghiệp vụ | 4 biến nghiệp vụ | 4 biến nghiệp vụ | **PASS** | Trích xuất chính xác 4 dynamic values cần cho downstream: `jwt_token`, `product_id`, `product_name`, `product_price`. |
| **I. Smoke Test Execution** | PASS | PASS | PASS | **PASS** | Chạy kiểm thử mẫu trên JMeter 5.6.3, các samplers thực thi tuần tự, nhận diện correlation và assertions thành công. |
| **J. Official Performance Execution** | NOT EXECUTED | NOT EXECUTED | NOT EXECUTED | **NOT EXECUTED** | Dành riêng cho Step 4 theo đúng quy trình; không chạy trước khi human review. |

---

## 2. Chi Tiết Các Samplers & Correlation Trong Từng Kịch Bản

Mỗi kịch bản gồm 6 HTTP Samplers tuần tự với cơ chế tương quan dữ liệu chính xác:

```
+-----------------------------------------------------------------------------------------------------------------------+
| Sampler Name                     | Method | Path                  | Correlation In / Out          | Assertion         |
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
| 01_POST_Register                 | POST   | /api/register         | In: ${vu_name}, ${vu_email},  | Code = 200        |
|                                  |        |                       |     ${vu_pass} (JSR223)       | Text: "User regis |
|                                  |        |                       |                               |  tered success..."|
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
| 02_POST_Login                    | POST   | /api/login            | In: ${vu_email}, ${vu_pass}   | Code = 200        |
|                                  |        |                       | Out: ${jwt_token}             | Text: "Login succ |
|                                  |        |                       |                               |  essful"          |
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
| 03_GET_Products_ReadHeavy        | GET    | /api/products         | Out: ${product_id},           | Code = 200        |
| [READ-HEAVY GROUP]               |        |                       |      ${product_name},         |                   |
|                                  |        |                       |      ${product_price}         |                   |
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
| 04_POST_Cart_Transactional       | POST   | /api/cart             | In: Bearer ${jwt_token},      | Code = 200        |
| [TRANSACTIONAL GROUP]            |        |                       |     ${product_id},            | Text: "Added to   |
|                                  |        |                       |     ${product_name/price}     |  cart"            |
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
| 05_GET_Cart_Verify               | GET    | /api/cart             | In: Bearer ${jwt_token}       | Code = 200        |
| [SUPPORTING VERIFY]              |        |                       |                               |                   |
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
| 06_POST_ForgotPassword_AuthHeavy | POST   | /api/forgot-password  | In: ${vu_email}               | Code = 200        |
| [AUTH-HEAVY GROUP]               |        |                       | (No downstream token needed)  | Text: "Mã đặt lại |
|                                  |        |                       |                               |  mật khẩu..."     |
+----------------------------------+--------+-----------------------+-------------------------------+-------------------+
```

---

## 3. Các Lỗi Đã Sửa (Fixed Issues in Step 3 Fix)

1. **Sửa lỗi XML Schema `assertionsResultsToSave`**:
   * *Nguyên nhân:* Thuộc tính `<assertionsResultsToSave>` trong `SampleSaveConfiguration` bị gán giá trị chuỗi `"false"`, gây lỗi `NumberFormatException: For input string: "false"` khi Apache JMeter nạp file.
   * *Đã sửa:* Đổi thành kiểu integer `0` (`<assertionsResultsToSave>0</assertionsResultsToSave>`), khớp chuẩn template `temp/EShop SUT Local - Purcharse Flow.jmx` của JMeter 5.6.3.

2. **Khắc phục lỗi file JMX bị chèn text hỏng**:
   * *Nguyên nhân:* File JMX cũ bị dán nhầm bảng markdown vào `testname` của sampler gây lỗi cú pháp parser XStream.
   * *Đã sửa:* Sử dụng `temp/generate_jmx.py` cập nhật để tái sinh tự động toàn bộ 3 file JMX chuẩn chỉnh.

3. **Cấu hình Workload Staircase cho Stress và Surge cho Spike bằng Native Components**:
   * *Nguyên nhân:* JMX cũ cấu hình Thread Group đơn lẻ không tạo ra đúng hình dạng bậc thang (Staircase) của Stress và tăng đột biến (Surge) của Spike.
   * *Đã sửa:*
     - **Stress**: Tạo 4 Thread Groups song song với start delay lệch pha (`0s, 50s, 100s, 150s`), mỗi stage 25 VUs, cộng dồn thành `25 → 50 → 75 → 100 VUs`.
     - **Spike**: Tạo 2 Thread Groups song song (Baseline 5 VUs chạy 170s; Surge 75 VUs bắt đầu tại 30s, ramp 10s, hold 60s, kết thúc tại 100s) tạo đỉnh tải 80 VUs và phục hồi về 5 VUs.

4. **Tối ưu hóa Correlation & Dọn dẹp CSV**:
   * *Nguyên nhân:* Trích xuất thừa `reset_token` và `auth_user_id` không dùng downstream; tồn tại `search_keywords.csv` không được workflow tham chiếu.
   * *Đã sửa:* Bỏ trích xuất thừa, chỉ giữ 4 biến thực sự cần; xóa file `search_keywords.csv` và giữ `data/user_profiles.csv` gọn nhẹ.

---

## 4. Các Điểm ASSUMPTION & UNKNOWN Cần Lưu Ý

1. **Môi trường kết nối SUT**:
   * Test Plan mặc định kết nối `http://localhost:3000`. Có thể override linh hoạt qua `-Jhost=<IP> -Jport=<PORT>` từ CLI.
2. **Hiện tượng `SQLITE_BUSY` trong Stress / Spike chính thức**:
   * Là một rủi ro tiềm năng (Possible / Unknown) có thể xảy ra khi backend SQLite chịu tải đồng thời ở mức 100 VUs. Trạng thái thực tế sẽ được quan sát và ghi nhận dựa trên evidence ở Step 4.

---

> 🎯 **Trạng thái:** Toàn bộ 3 kịch bản `.jmx` và dữ liệu CSV đã được sửa đổi và kiểm tra tương thích thành công với Apache JMeter 5.6.3.  
> **Không thực hiện official performance execution ở Step 3 Fix.**  
> **Dừng lại để Human Review!**
