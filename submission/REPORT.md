# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm:
- Repository URL:
- Commit SHA cuối:
- Thành viên và vai trò:
  - Nguyễn Văn Trọng — Logging & PII (CP1)
  - Nguyễn Tuấn Hùng - Metrics, Traces, Dashboard & Alerts (Checkpoint CP2)
  - Phạm Tiến Hưng - Challenge: Điều tra Incident (Checkpoint CP3)

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: **100/100**
- Tổng số traces: **ít nhất 10** (ảnh evidence ghi nhận 40 observations, gồm 20 generation và 20 span)
- Số PII leak còn lại: 0
- Dashboard: **Langfuse → My Project → Dashboards → SYSTEM OBSERVABILITY DASHBOARD**

## 3. Logging và tracing

### Evidence correlation ID

Xem file: `submission/evidence/cp1_log_correlation_id.txt`

Mỗi request được gán một `correlation_id` duy nhất dạng `req-<8 hex>`.
Hai dòng log `request_received` và `response_sent` cùng chung ID, chứng minh truy vết end-to-end:

```json
{"event": "request_received", "correlation_id": "req-a649877a", "session_id": "s01", ...}
{"event": "response_sent",    "correlation_id": "req-a649877a", "session_id": "s01", ...}
```

### Evidence PII redaction

Xem file: `submission/evidence/cp1_pii_redaction.txt`

| Dữ liệu gốc (input) | Trong log (sau scrub) |
|---|---|
| `student@vinuni.edu.vn` | `[REDACTED_EMAIL]` |
| `0987654321` | `[REDACTED_PHONE_VN]` |
| `4111 1111 1111 1111` | `[REDACTED_CREDIT_CARD]` |

Kiểm chứng: chạy `Select-String "@" data/logs.jsonl` trả về rỗng → không lộ email thô.

### Evidence trace waterfall

Xem ảnh: `submission/evidence/Screenshot một trace waterfall.png`

Trace ID: `f8ffaec311e3586bcedab7a33394e897`.

Waterfall cho thấy một trace `run` gồm một span gốc và một generation con, với thời lượng tổng **3.18 s** và chi phí **$0.001956**. Trace có metadata `session_id=s02`, `env=default` và `user_id` đã được hash.

### Giải thích một span đáng chú ý

Generation `run` là span chiếm gần như toàn bộ thời gian trace (3.18 s). Log View của trace thể hiện metadata như `doc_count`, `query_preview`, `prompt_name`, `prompt_label`, `prompt_version` và `prompt_source`. Khi latency tăng, nhóm sẽ bắt đầu từ span này, lấy trace ID/correlation ID rồi đối chiếu log để xác định request và bước xử lý gây chậm.


## 4. Prompt versioning

- Prompt name:
- Version/label baseline:
- Version/label candidate:
- Trace ID của mỗi version:
- Bằng chứng đổi label hoặc rollback:

## 5. Dashboard, SLO và alerts

- **Kết quả `validate_dashboard.py`:** `HỢP LỆ: 6/6 panel có trong dashboard contract.`
- **Evidence dashboard:** `submission/evidence/Dashboard.png`. Ảnh dashboard Langfuse có time range **Past 1 hour** và thể hiện đủ sáu nhóm chỉ số: latency P50/P95/P99, traffic, error rate/breakdown, cost, input/output tokens và average quality.

### SLO đã chọn

SLO được khai báo tại `config/slo.yaml`, dùng cửa sổ đánh giá 28 ngày và nhất quán với threshold của dashboard.

| SLI | Objective | Target | Lý do |
|---|---:|---:|---|
| `latency_p95_ms` | P95 ≤ 3,000 ms | 99.5% | Phản hồi quá 3 giây ảnh hưởng trực tiếp tới trải nghiệm người dùng. |
| `error_rate_pct` | ≤ 2% | 99.0% | Giữ phần lớn request nhận được phản hồi hợp lệ. |
| `daily_cost_usd` | ≤ $2.50/ngày | 100.0% | Kiểm soát chi phí vận hành trong ngân sách lab. |
| `quality_score_avg` | ≥ 0.75 | 95.0% | Duy trì chất lượng phản hồi ở mức chấp nhận được. |

### Alert rules và runbook

Alert rules nằm tại `config/alert_rules.yaml`; hướng dẫn phản ứng tương ứng nằm tại `docs/alerts.md`. Cả ba alert đều dựa trên triệu chứng người dùng/SLO, không phụ thuộc tên implementation nội bộ.

| Alert | Severity | Điều kiện kích hoạt | Owner |
|---|---|---|---|
| `high_latency_p95` | warning | `latency_p95 > 3000ms for 5 minutes` | on-call-engineer |
| `elevated_error_rate` | critical | `error_rate_pct > 5 for 3 minutes` | on-call-engineer |
| `cost_budget_exceeded` | warning | `daily_cost_usd > 2.5` | team-lead |

Mỗi runbook có ba bước kiểm tra đầu tiên theo luồng **Dashboard → Langfuse trace → log theo correlation ID**, cùng mitigation tạm thời và owner chịu trách nhiệm.

## 6. Điều tra challenge

- Challenge ID: `day13-k4-observability-v1`
- Triệu chứng từ metrics: P95 Latency tăng vọt lên mức ~2.7s (vượt ngưỡng SLO, kích hoạt alert `high_latency_p95`).
- Trace ID liên quan: (Có thể truy xuất trace trên Langfuse tương ứng với ID bên dưới).
- Log line/correlation ID liên quan: `req-7fdf40a3` (Dòng log `response_sent` ghi nhận `latency_ms`: 2667).
- Root cause: Xem trên Langfuse Trace Waterfall thấy span `retrieve` (gọi ở file `mock_rag.py`) bị treo mất 2.5s do sự cố `rag_slow` (`time.sleep(2.5)`).
- Fix action: Loại bỏ lệnh delay/sửa lỗi trong code hàm `retrieve()` tại `app/mock_rag.py` (tắt cờ incident).
- Preventive measure: Thiết lập `timeout` tối đa (vd: 500ms) cho các truy vấn Vector Database/RAG. Nếu vượt quá thời gian, dùng fallback response/cache thay vì để request bị treo toàn hệ thống.

## 7. Đóng góp cá nhân

### Nguyễn Văn Trọng — Logging & PII (CP1)

**Commit SHA:** `ee7d81d`, `e54a876`

**Phần việc đã thực hiện:**

| # | Công việc | File đã sửa | Mô tả |
|---|-----------|-------------|-------|
| 1 | Correlation ID Middleware | `app/middleware.py` | `clear_contextvars()` để tránh leak context, tạo ID dạng `req-<8 hex>`, bind vào structlog, trả qua response header |
| 2 | Log Enrichment | `app/main.py` | `bind_contextvars(user_id_hash, session_id, feature, model, env)` trong hàm `chat()` |
| 3 | PII Scrubbing | `app/logging_config.py` | Bật `scrub_event` processor, mở rộng quét toàn bộ field string/dict thay vì chỉ payload |
| 4 | Thêm PII patterns | `app/pii.py` | Thêm regex cho passport (`[A-Z]\d{7,8}`) và địa chỉ VN |
| 5 | Liên kết Log ↔ Trace | `app/agent.py` | Đính kèm `correlation_id` vào metadata Langfuse trace |
| 6 | Error rate metric | `app/metrics.py` | Tính `error_rate_pct` trong `snapshot()` phục vụ dashboard |
| 7 | Exception handler | `app/main.py` | Giữ `x-request-id` trong response khi lỗi 500 |
| 8 | Cập nhật test | `tests/test_agent_prompt_trace.py` | Thêm `correlation_id` vào expected metadata |

**Điều đã học:**

- `clear_contextvars()` bắt buộc vì ASGI server tái sử dụng task/thread — nếu không clear, context cũ leak sang request mới gây gán nhầm user.
- Thứ tự processor trong structlog rất quan trọng: `scrub_event` phải nằm **sau** `TimeStamper` và **trước** `JsonlFileProcessor` để PII được che trước khi ghi file.
- Dùng `hash_user_id()` (SHA-256) thay vì lưu user_id gốc — cho phép nhóm log theo user mà không lộ danh tính.

### Nguyễn Tuấn Hùng — Metrics, Traces, Dashboard & Alerts (CP2)

**Commit SHA:** `8ec4aab`

**Phần việc đã thực hiện:**

| # | Công việc | File/evidence | Mô tả |
|---|---|---|---|
| 1 | Dashboard Langfuse | `submission/evidence/Dashboard.png` | Tạo dashboard `SYSTEM OBSERVABILITY DASHBOARD` theo time range 1 giờ, thể hiện đủ 6 nhóm chỉ số bắt buộc. |
| 2 | Dashboard specification | `docs/dashboard-spec.md` | Ghi rõ công cụ Langfuse, nguồn dữ liệu, đơn vị, time range, refresh và threshold của 6 nhóm chỉ số. |
| 3 | SLO | `config/slo.yaml` | Đặt SLO latency, error rate, daily cost và quality phù hợp contract dashboard. |
| 4 | Alert rules | `config/alert_rules.yaml` | Khai báo ba alert symptom-based cho chậm phản hồi, tỷ lệ lỗi tăng và vượt ngân sách. |
| 5 | Alert runbook | `docs/alerts.md` | Viết mức độ ảnh hưởng, ba bước kiểm tra đầu tiên, mitigation và owner cho từng alert. |
| 6 | Trace evidence | `submission/evidence/Screenshot danh sách ≥ 10 traces.png`, `submission/evidence/Screenshot một trace waterfall.png` | Lưu danh sách traces và waterfall để điều tra theo chuỗi Metrics → Traces → Logs. |

**Điều đã học:**

- P95 phù hợp hơn trung bình để phát hiện nhóm request chậm gây ảnh hưởng xấu tới người dùng.
- Alert tốt phải nêu triệu chứng, mức độ, thời gian duy trì, owner và runbook; không chỉ báo tên một component nội bộ.
- Luồng điều tra hiệu quả là xác nhận triệu chứng trên dashboard, mở trace bất thường trong Langfuse, sau đó dùng `correlation_id` để chứng minh nguyên nhân bằng log.

### Phạm Tiến Hưng — Challenge: Điều tra Incident (CP3)

**Commit SHA:** (Cập nhật sau khi push)

**Phần việc đã thực hiện:**

| # | Công việc | File/evidence | Mô tả |
|---|---|---|---|
| 1 | Cài đặt môi trường | `scripts/` | Xử lý lỗi thư viện thiếu (uvicorn, structlog) để khởi chạy API. |
| 2 | Kích hoạt sự cố | `config/challenge.json` | Chạy `inject_incident.py` để inject sự cố `rag_slow` vào hệ thống mô phỏng. |
| 3 | Sinh tải (Load Test) | `scripts/load_test.py` | Chạy bộ test với `--challenge` để đo lường độ trễ của hệ thống. |
| 4 | Truy vết Log | `data/logs.jsonl` | Dùng PowerShell lọc ra `correlation_id` (vd: `req-7fdf40a3`) có `latency_ms` > 2.6s. |
| 5 | Phân tích Root Cause | `app/mock_rag.py` | Dựa vào Trace, phát hiện span `retrieve` bị delay 2.5s do sự cố `rag_slow`. |
| 6 | Tổng hợp Báo cáo | `submission/REPORT.md` | Ghi nhận nguyên nhân, hướng khắc phục và biện pháp phòng ngừa (thêm timeout, fallback). |

**Điều đã học:**

- Luồng Observability (Metrics → Traces → Logs) giúp thu hẹp phạm vi từ "có gì đó sai sai" thành "hàm A dòng B bị lỗi".
- Các dịch vụ bên thứ ba (như Vector DB) không bao giờ nên chặn (block) toàn bộ API mà phải có cơ chế Timeout và Fallback an toàn.
