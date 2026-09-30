# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert mẫu để tham khảo

Ví dụ dưới đây minh họa mức độ cụ thể cần có. Học viên không cần copy nguyên, nhưng ba alert trong bài nộp nên rõ ràng tương tự: điều kiện là gì, kéo dài bao lâu, ảnh hưởng tới user ra sao và người trực cần kiểm tra gì trước.

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms`
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` trong 5 phút
- Ảnh hưởng tới người dùng: người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard latency để xác nhận P95/P99 và khoảng thời gian tăng.
  2. Lọc `data/logs.jsonl` trong khoảng đó, lấy một `correlation_id` có `latency_ms` cao.
  3. Mở trace cùng `correlation_id` trên Langfuse, so sánh các span chính để xác định bước nào bất thường.
- Mitigation tạm thời: dựa trên evidence thực tế để rollback prompt, khôi phục cấu hình liên quan, tắt practice scenario hoặc giảm tải khi demo.
- Owner: `student-<MSSV>`

## Alert 1

- Tên: `HighLatencyP95`
- Severity: `warning`
- Duration: `5m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: latency P95 của `response_sent.latency_ms` <= 3000ms
- Điều kiện và thời gian duy trì: `p95(latency_ms) > 3000ms` duy trì trong 5 phút
- Ảnh hưởng tới người dùng: Người dùng phải chờ lâu hơn trước khi nhận câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Mở dashboard Latency xác nhận mức P95/P99 và khoảng thời gian bắt đầu tăng.
  2. Lọc `data/logs.jsonl` theo khoảng thời gian đó, lấy một `correlation_id` bị latency cao.
  3. Mở Trace trên Langfuse có cùng `correlation_id` để kiểm tra span nào (retrieval hay generation) bị kéo dài.
- Mitigation tạm thời: Rollback prompt version nếu mới promote prompt, hoặc ngắt các tiến trình cào data/rag slow.
- Owner: `student-02748`

## Alert 2

- Tên: `HighErrorRate`
- Severity: `critical`
- Duration: `2m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỷ lệ request thất bại `request_failed` / `request_received` <= 2%
- Điều kiện và thời gian duy trì: `error_rate_pct > 5%` duy trì trong 2 phút
- Ảnh hưởng tới người dùng: Người dùng nhận phản hồi lỗi HTTP 500
- Ba bước kiểm tra đầu tiên:
  1. Mở panel Errors trên Dashboard kiểm tra error_type (ví dụ RuntimeError).
  2. Lọc `data/logs.jsonl` tìm log line `event="request_failed"`, lấy `correlation_id`.
  3. Kiểm tra Trace trên Langfuse để xem exception phát sinh tại tool hay API client.
- Mitigation tạm thời: Khôi phục dịch vụ phía sau (vector store), restart API instance hoặc disable incident practice.
- Owner: `student-02748`

## Alert 3

- Tên: `RetrievalFailureRate`
- Severity: `warning`
- Duration: `3m`
- Kênh thông báo: Slack `#k4-l3b-alerts`
- SLI/SLO liên quan: Tỷ lệ truy vấn retrieval thành công >= 90%
- Điều kiện và thời gian duy trì: `retrieval_success_rate_pct < 90%` duy trì trong 3 phút
- Ảnh hưởng tới người dùng: Khả năng trả lời đúng context giảm, câu trả lời bị rơi vào fallback chung
- Ba bước kiểm tra đầu tiên:
  1. Kiểm tra panel Errors/Retrieval success rate trên Dashboard.
  2. Lọc log `data/logs.jsonl` kiểm tra các log có `tool_name="retrieval"` và `tool_success=False`.
  3. Tra cứu Trace ID tương ứng trên Langfuse để xem span `retrieval` bị lỗi timeout hay kết nối.
- Mitigation tạm thời: Kiểm tra kết nối tới Vector DB/Corpus store, restart service retrieval.
- Owner: `student-02748`
