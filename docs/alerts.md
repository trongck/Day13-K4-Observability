# Alert rules và runbook

Các alert dưới đây dựa trên triệu chứng người dùng hoặc SLO. Khi điều tra, luôn bắt đầu từ dashboard, sau đó mở trace trong Langfuse và cuối cùng đối chiếu log theo `correlation_id`.

## Alert 1

- **Tên:** `high_latency_p95`
- **Severity:** warning
- **SLI/SLO liên quan:** `latency_p95_ms`; mục tiêu P95 ≤ 3,000 ms cho 99.5% thời gian trong cửa sổ SLO 28 ngày.
- **Điều kiện kích hoạt:** `latency_p95 > 3000ms for 5 minutes`.
- **Ảnh hưởng tới người dùng:** Một phần người dùng phải chờ quá 3 giây để nhận phản hồi; họ có thể thấy ứng dụng chậm hoặc gửi lại request.
- **Ba bước kiểm tra đầu tiên:**
  1. Xác nhận trên dashboard rằng P95 vượt 3,000 ms liên tục 5 phút; so sánh traffic, error rate và latency P50/P99 trong cùng khoảng thời gian để xác định phạm vi ảnh hưởng.
  2. Mở các trace chậm trong Langfuse của khoảng thời gian đó, sắp xếp theo duration và ghi lại trace ID/span có thời gian cao bất thường.
  3. Lấy `correlation_id` từ trace rồi tìm các log cùng ID trong `data/logs.jsonl`; đối chiếu event và thời gian để khoanh vùng bước xử lý chậm.
- **Mitigation tạm thời:** Giảm tải bằng cách giới hạn request đồng thời hoặc áp dụng rate limit; chuyển tạm sang cấu hình phản hồi ngắn/fallback đã được kiểm chứng. Theo dõi P95 sau mỗi 5 phút và gỡ biện pháp khi SLO ổn định.
- **Owner:** on-call-engineer

## Alert 2

- **Tên:** `elevated_error_rate`
- **Severity:** critical
- **SLI/SLO liên quan:** `error_rate_pct`; SLO là error rate ≤ 2% cho 99.0% thời gian trong cửa sổ 28 ngày.
- **Điều kiện kích hoạt:** `error_rate_pct > 5 for 3 minutes`.
- **Ảnh hưởng tới người dùng:** Ít nhất 5% request không nhận được phản hồi hợp lệ hoặc phải thử lại; trải nghiệm của người dùng bị gián đoạn trực tiếp.
- **Ba bước kiểm tra đầu tiên:**
  1. Xác nhận error rate trên dashboard vượt 5% liên tục 3 phút và kiểm tra traffic để loại trừ trường hợp mẫu dữ liệu quá nhỏ.
  2. Xem panel error breakdown để xác định `error_type` nào tăng nhiều nhất; mở trace lỗi tương ứng trong Langfuse để lấy trace ID và thông tin request bị ảnh hưởng.
  3. Dùng `correlation_id` của một trace lỗi để tìm log `request_failed` trong `data/logs.jsonl`; so sánh các lỗi cùng loại và thời điểm bắt đầu tăng.
- **Mitigation tạm thời:** Chuyển về prompt/configuration đã ổn định gần nhất hoặc bật fallback response cho request lỗi; giới hạn retry để tránh làm số lỗi và tải tăng thêm. Thông báo tình trạng cho team khi alert còn kích hoạt.
- **Owner:** on-call-engineer

## Alert 3

- **Tên:** `cost_budget_exceeded`
- **Severity:** warning
- **SLI/SLO liên quan:** `daily_cost_usd`; ngân sách là tổng chi phí mỗi ngày ≤ 2.50 USD, target 100% trong cửa sổ SLO 28 ngày.
- **Điều kiện kích hoạt:** `daily_cost_usd > 2.5`.
- **Ảnh hưởng tới người dùng:** Ngân sách vận hành hàng ngày bị vượt; nếu không xử lý, nhóm có thể phải giới hạn dịch vụ, làm giảm khả năng sử dụng của người dùng về sau.
- **Ba bước kiểm tra đầu tiên:**
  1. Xác nhận tổng `daily_cost_usd` và biểu đồ Cost over time trên dashboard; so sánh với traffic và tổng token để xem chi phí tăng do số request hay do chi phí trên từng request.
  2. Lọc trace trong Langfuse theo khoảng thời gian chi phí tăng, xác định các trace có token/cost cao và ghi lại trace ID, model hoặc feature liên quan.
  3. Đối chiếu các `correlation_id` này trong `data/logs.jsonl` để xác nhận `tokens_in`, `tokens_out` và `cost_usd`, rồi xác định mẫu request tốn chi phí bất thường.
- **Mitigation tạm thời:** Áp dụng rate limit hoặc quota tạm thời cho traffic gây chi phí cao; ưu tiên cấu hình/prompt ngắn hơn hoặc fallback ít tốn chi phí hơn cho request không quan trọng. Chỉ gỡ hạn chế sau khi chi phí quay về dưới ngưỡng theo dõi.
- **Owner:** team-lead
