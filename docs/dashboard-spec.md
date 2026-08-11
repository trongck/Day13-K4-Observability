# Dashboard specification — Day 13 AI Observability

## Công cụ và cấu hình chung

- **Công cụ sử dụng:** Langfuse.
- **Dashboard:** `SYSTEM OBSERVABILITY DASHBOARD`.
- **Khoảng thời gian mặc định:** 60 phút gần nhất.
- **Tự làm mới:** mỗi 30 giây (nếu workspace Langfuse hỗ trợ refresh tự động).
- **Nguồn dữ liệu chuẩn:** các sự kiện đã ghi cho request/response, đối chiếu với `data/logs.jsonl` theo contract tại `config/dashboard.yaml`.
- **Quy ước cảnh báo:** đường threshold/SLO hiển thị trên panel; màu xanh đạt ngưỡng, màu đỏ khi vi phạm.

## Các panel chính

Dashboard gồm đúng 6 panel quan trọng sau:

| # | Tên panel | Dữ liệu và phép tính | Đơn vị | Threshold / SLO line |
|---|---|---|---|---|
| 1 | **Latency percentiles** | Với event `response_sent`, tính P50, P95 và P99 của `latency_ms`; hiển thị bằng biểu đồ theo thời gian hoặc ba giá trị chính. | ms | P95 ≤ **3,000 ms** |
| 2 | **Request traffic** | Đếm event `request_received` theo từng phút; hiển thị request count hoặc request/phút. | requests/minute | Traffic ≥ **1 request/phút** |
| 3 | **Error rate and breakdown** | `request_failed / request_received × 100` và bảng/biểu đồ phân loại theo `error_type`. | % | Error rate ≤ **2%** |
| 4 | **Cost over time** | Với event `response_sent`, cộng `cost_usd` theo từng phút và hiển thị tổng trong cửa sổ thời gian. | USD | Total cost ≤ **$2.50** |
| 5 | **Input and output tokens** | Với event `response_sent`, cộng riêng `tokens_in` và `tokens_out`. | tokens | Tổng token ≤ **50,000** |
| 6 | **Quality proxy** | Với event `response_sent`, tính trung bình của `quality_score`. | score (0–1) | Quality average ≥ **0.75** |

## Mapping dữ liệu

| Panel | Event | Field |
|---|---|---|
| Latency percentiles | `response_sent` | `latency_ms` |
| Request traffic | `request_received` | `event` |
| Error rate and breakdown | `request_received`, `request_failed` | `error_type` |
| Cost over time | `response_sent` | `cost_usd` |
| Input and output tokens | `response_sent` | `tokens_in`, `tokens_out` |
| Quality proxy | `response_sent` | `quality_score` |

## Evidence

- Ảnh dashboard Langfuse được lưu trong `submission/evidence/` và phải nhìn rõ: tên các panel, time range **Last 1 hour**, đơn vị đo và các threshold/SLO line.
- Trước khi nộp evidence, kiểm tra contract bằng:

```bash
python scripts/validate_dashboard.py
```

Kết quả mong đợi: `HỢP LỆ: 6/6 panel có trong dashboard contract.`
