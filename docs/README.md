# HW05 - AI-Assisted Performance Testing

> Lê Thanh Phong — MSSV: 23127452
> **Github:** [Public GitHub Repository](https://github.com/ltpisme/CSC15003_HW5)
> **Video Demo Task 1:** [Task 1 Demo](https://youtu.be/uO4wviaSuKY)
> **Video Demo Agent Skill:** [Agent Skill Demo](https://youtu.be/WCK8AW-1zeE)

## 1. Báo Cáo Tóm Tắt Kiểm Thử (Test Summary Report)

### Bảng Kết Quả Thực Nghiệm 4 Kịch Bản

| Kịch Bản (Test Type)                            | Tệp Kịch Bản (Test Plan)                                                                                                                      | Dữ Liệu Đầu Vào (Dataset)                           | Kết Quả Thực Nghiệm (Result)                                                                                                                  | Bằng Chứng Minh Họa (Evidence)                                                                                    |
| :------------------------------------------------ | :----------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------------------------------- |
| **Load Testing***(Tải danh định)*        | [`jmeter/23127452_Load_20260902.jmx`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/jmeter/23127452_Load_20260902.jmx)                   | `data/user_profiles.csv`(20 VUs, 224s)                 | • Samples:**8,290**• Error: **0.00%**• Throughput: **37.10 req/s**• Mean: **6.21 ms** \| P95: **13.0 ms**       | •`results/load/raw.jtl`• `results/load/html/`• `evidence/load/load_jmeter_resource.png`                     |
| **Stress Testing***(Bão hòa tải)*        | [`jmeter/23127452_Stress_20260902.jmx`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/jmeter/23127452_Stress_20260902.jmx)               | `data/user_profiles.csv`(Staircase: 25→100 VUs, 300s) | • Samples:**64,231**• Error: **0.00%**• Throughput: **214.34 req/s**• Mean: **140.16 ms** \| P95: **876.95 ms** | •`results/stress/raw.jtl`• `results/stress/html/`• `evidence/stress/stress_jmeter_resource.png`             |
| **Spike Testing***(Đột biến tải)*       | [`jmeter/23127452_Spike_20260902.jmx`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/jmeter/23127452_Spike_20260902.jmx)                 | `data/user_profiles.csv`(5→80→5 VUs, 170s)           | • Samples:**21,470**• Error: **0.00%**• Throughput: **127.26 req/s**• Mean: **167.69 ms** \| P95: **892.00 ms** | •`results/spike/raw.jtl`• `results/spike/html/`• `evidence/spike/spike_jmeter_resource.png`                 |
| **Endurance Testing***(Độ bền 15 phút)* | [`jmeter/23127452_Load_20260902.jmx`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/jmeter/23127452_Load_20260902.jmx)`-Jduration=900` | `data/user_profiles.csv`(20 VUs, 900s)                 | • Samples:**34,677**• Error: **0.00%**• Throughput: **38.59 req/s**• Mean: **7.86 ms** \| P95: **26.0 ms**      | •`results/endurance/raw.jtl`• `results/endurance/html/`• `evidence/endurance/endurance_jmeter_resource.png` |

### Ngưỡng Chịu Tải Phần Cứng Bền Vững

* **Thông lượng ổn định bền vững tối đa (Sustained Throughput):** **`38.59 samples/sec`** ($\sim 6.43\text{ E2E transactions/sec}$).
* **Trần tiêu thụ bộ nhớ (Memory Ceiling):** **`~85 MB Node.js RSS`** (duy trì phẳng, không có rò rỉ bộ nhớ).
* **Độ trễ P95 ổn định:** **`<= 26.0 ms`**.
* **Tỷ lệ lỗi:** **`0.00%`** qua 15 phút (900 giây) ngâm tải liên tục.

## 2. Tổng Hợp Lỗi Hệ Thống & Điểm Nghẽn (Bug Summary)

Chi tiết trích xuất từ tài liệu [`temp/AI/13_bug_report.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/temp/AI/13_bug_report.md):

|    Mã Lỗi (ID)    | Tên Lỗi (Action + Actual Result)                                                          | Mức Độ (Severity) | Vị Trí Mã Nguồn (Location)      |
| :------------------: | :------------------------------------------------------------------------------------------ | :------------------: | :---------------------------------- |
| **`BUG-01`** | `Submit login failure increments attempt counter by 2 instead of 1 causing early lockout` |        Medium        | `eshop/backend/server.js:54-58`   |
| **`BUG-02`** | `Execute search query returns raw HTML error page and enables SQL injection`              |       Critical       | `eshop/backend/server.js:144-149` |
| **`BUG-03`** | `Request product detail with even ID returns price as string instead of number`           |        Medium        | `eshop/backend/server.js:162`     |
| **`BUG-04`** | `Request order cancellation on active order allows canceling non-pending order`           |        Medium        | `eshop/backend/server.js:329-331` |

* **Điểm nghẽn hiệu năng (Performance Bottlenecks):**
  1. *SQLite Single-Writer Lock Contention:* Gây suy thoái độ trễ `POST /api/register` lên P95 = 1,760 ms dưới tải 100 VUs.
  2. *Full Table Scan:* Bảng `users` thiếu chỉ mục trên cột `email`.

## 3. Bảng Tự Đánh Giá Điểm

Tuân thủ đúng định dạng mẫu đánh giá tại Section 15 của `ref/HW05.md`:

|       STT (No.)       | Tiêu Chí Đánh Giá (Criteria)                                                  | Điểm Tối Đa (Max Grade) | Điểm Tự Đánh Giá (Self-Assessed Grade) | Minh Chứng / Trạng Thái Hoàn Thành                                                                     |
| :-------------------: | :--------------------------------------------------------------------------------- | :-------------------------: | :------------------------------------------: | :---------------------------------------------------------------------------------------------------------- |
|      **1**      | Task 1 — Load testing                                                             |             30             |                 **30**                 | Đầy đủ JMX, JTL, HTML Dashboard, P95 = 13.0 ms, Error = 0.00%, ảnh`htop`.                            |
|      **2**      | Task 1 — Stress testing                                                           |             20             |                 **20**                 | Staircase 4 cấp (25→100 VUs), JTL, HTML, P95 = 876.95 ms, ảnh`htop`.                                   |
|      **3**      | Task 1 — Spike testing                                                            |             20             |                 **20**                 | Surge 80 VUs & recovery, JTL, HTML, P95 = 892.00 ms, ảnh`htop`.                                          |
|      **4**      | Task 2 — AI analysis + misinterpretation hunt (with correct values from raw logs) |             10             |                 **10**                 | Đối chiếu chéo chỉ ra sai lệch P90/P95 Stress của AI; thẩm định 9 đề xuất tối ưu.            |
|      **5**      | Task 3 — Continuous Performance Testing proposal (G9.6 Disrupt)                   |             10             |                 **10**                 | Sơ đồ Mermaid Flowchart 5 giai đoạn, 3 cổng chặn PR ($\Delta P95 > 15\%$), phân tích trade-offs. |
|      **6**      | Agent Skills                                                                       |             10             |                 **10**                 | Đã xây dựng 2 Agent Skills hoàn chỉnh với 7 knowledge files và 3 templates.                         |
| **TỔNG CỘNG** | **TOTAL**                                                                    |        **100**        |                **100**                | **100/100 (Sẵn sàng nộp sau khi quay video và đóng gói ZIP)**                                  |

## 4. Tóm Tắt Kiểm Định AI

* **Tuyên bố AI:** *"I use AI tools for the following tasks: E2E workflow design, JMeter test plan generation, test execution audit, log analysis, and CI/CD continuous performance pipeline proposal."*
* **Công cụ sử dụng:** `Antigravity - Gemini 3.7 Flash`, `ChatGPT`, `Apache JMeter 5.6.3`, `htop`.
* **Nhật ký tương tác:** Gồm 6 phiên làm việc chính thức ghi nhận đầy đủ theo cấu trúc `template/AI_log.md` tại [`docs/AI_Audit.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/docs/AI_Audit.md).
* **Phê bình AI (AI Critique):** Đúng **258 từ** (nằm trong khoảng 200–300 từ) trả lời trọn vẹn 3 câu hỏi bắt buộc về các sai lệch số liệu, đề xuất ảo tưởng của AI và bài học kiểm chứng độc lập từ dữ liệu gốc.

## 5. Bộ Công Cụ Kỹ Năng Tự Động Hóa

Đã trích xuất và chuẩn hóa quy trình kiểm thử hiệu năng thành 2 Agent Skills độc lập, có tính xác định và tái sử dụng tại thư mục [`agent/skills/`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/):

### 1. [`performance-test-generation`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/SKILL.md)

* **Mục tiêu:** Tự động phân tích đặc tả REST API, thiết kế 4 kịch bản kiểm thử hiệu năng, sinh mã XML JMeter 5.6.3 (`*.jmx`) hoàn chỉnh, tham số hóa dữ liệu CSV và tạo script lệnh thực thi headless CLI.
* **Đầu vào (Input):** REST API Specification (OpenAPI / Swagger / Markdown), Yêu cầu kiểm thử & mục tiêu SLOs.
* **Đầu ra (Output):** Các tệp kịch bản `.jmx`, bộ dữ liệu `data/*.csv`, tài liệu lệnh thực thi `execution_commands.md`.
* **Knowledge & Templates:** [`performance-metrics.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/knowledge/performance-metrics.md), [`api-analysis.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/knowledge/api-analysis.md), [`jmeter-patterns.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/knowledge/jmeter-patterns.md), [`test-types.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/knowledge/test-types.md), [`jmx-plan-template.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/templates/jmx-plan-template.md), [`execution-commands-template.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-test-generation/templates/execution-commands-template.md).

### 2. [`performance-result-analysis`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-result-analysis/SKILL.md)

* **Mục tiêu:** Phân tích dữ liệu thực nghiệm thô (`raw.jtl`), đối chiếu thống kê (`statistics.json`), thẩm định bằng chứng tài nguyên (`evidence/*.png`), phát hiện sai lệch và xuất báo cáo kiểm toán hiệu năng chuẩn mực.
* **Đầu vào (Input):** `raw.jtl`, `statistics.json`, `evidence/*.png`, Hardware spec.
* **Đầu ra (Output):** Báo cáo phân tích hiệu năng có cấu trúc (`performance_analysis_report.md`).
* **Knowledge & Templates:** [`performance-metrics.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-result-analysis/knowledge/performance-metrics.md), [`result-analysis.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-result-analysis/knowledge/result-analysis.md), [`evidence-analysis.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-result-analysis/knowledge/evidence-analysis.md), [`performance-analysis-template.md`](file:///home/ltp/Code/Course/Testing/HW/CSC15003_HW5/agent/skills/performance-result-analysis/templates/performance-analysis-template.md).

## 6. Danh Mục Tài Liệu Bàn Giao

```text
.
├── README.md                                    # Báo cáo tổng quan & Bảng tự đánh giá điểm
├── docs/
│   ├── report.md                                # Báo cáo chính (Markdown)
│   ├── report.pdf                               # Báo cáo chính (PDF)
│   ├── AI_Audit.md                              # Nhật ký kiểm định AI & AI Critique (Markdown)
│   ├── AI_Audit.pdf                             # Nhật ký kiểm định AI & AI Critique (PDF)
│   └── AI_Audit/                                # Thư mục chứa artifacts kỹ thuật của AI
├── jmeter/
│   ├── 23127452_Load_20260902.jmx               # Kịch bản Load Testing (View Results Tree)
│   ├── 23127452_Stress_20260902.jmx             # Kịch bản Stress Testing (Aggregate Report)
│   ├── 23127452_Spike_20260902.jmx              # Kịch bản Spike Testing (Summary Report)
│   └── data/
│       └── user_profiles.csv                    # Dữ liệu CSV tham số hóa
├── results/
│   ├── load/ (raw.jtl, html/)                   # Kết quả kịch bản Load
│   ├── stress/ (raw.jtl, html/)                 # Kết quả kịch bản Stress
│   ├── spike/ (raw.jtl, html/)                  # Kết quả kịch bản Spike
│   └── endurance/ (raw.jtl, html/)              # Kết quả kịch bản Endurance 15 phút
├── evidence/
│   ├── hardware/hardware_hostname.png           # Minh chứng cấu hình phần cứng & Hostname ltp
│   ├── load/load_jmeter_resource.png            # Ảnh chụp JMeter CLI + htop bài Load
│   ├── stress/stress_jmeter_resource.png        # Ảnh chụp JMeter CLI + htop bài Stress
│   ├── spike/spike_jmeter_resource.png          # Ảnh chụp JMeter CLI + htop bài Spike
│   └── endurance/endurance_jmeter_resource.png  # Ảnh chụp JMeter CLI + htop bài Endurance
└── agent/skills/                                # Bộ công cụ Agent Skills tái sử dụng
    ├── performance-test-generation/
    └── performance-result-analysis/
```
