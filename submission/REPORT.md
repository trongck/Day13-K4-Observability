# Báo cáo Day 13 Observability

## 1. Thông tin nhóm

- Tên nhóm:
- Repository URL:
- Commit SHA cuối:
- Thành viên và vai trò:
  - Nguyễn Văn Trọng — Logging & PII (CP1)

## 2. Kết quả kỹ thuật

- Điểm `validate_logs.py`: **100/100**
- Tổng số traces: 10
- Số PII leak còn lại: 0
- Link/đường dẫn dashboard: http://127.0.0.1:8000/metrics

## 3. Logging và tracing

### Evidence correlation ID

Xem file: `submission/evidence/cp1_log_correlation_id.txt`

Mỗi request được gán một `correlation_id` duy nhất dạng `req-<8 hex>`.
Hai dòng log `request_received` và `response_sent` cùng chung ID, chứng minh truy vết end-to-end:

```json
{"event": "request_received", "correlation_id": "req-ce9a43be", "session_id": "s01", ...}
{"event": "response_sent",    "correlation_id": "req-ce9a43be", "session_id": "s01", ...}
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



### Giải thích một span đáng chú ý



## 4. Prompt versioning

- Prompt name:
- Version/label baseline:
- Version/label candidate:
- Trace ID của mỗi version:
- Bằng chứng đổi label hoặc rollback:

## 5. Dashboard, SLO và alerts

- Kết quả `validate_dashboard.py`:
- Evidence dashboard:
- SLO đã chọn và lý do:
- Alert rules và runbook:

## 6. Điều tra challenge

- Challenge ID:
- Triệu chứng từ metrics:
- Trace ID liên quan:
- Log line/correlation ID liên quan:
- Root cause:
- Fix action:
- Preventive measure:

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
