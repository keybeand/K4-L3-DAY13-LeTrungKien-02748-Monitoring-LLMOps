# Báo cáo cá nhân — K4-L3B Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Lê Trung Kiên
- **MSSV:** 02748
- **Lớp:** K4-L3B
- **Repository URL:** https://github.com/keybeand/K4-L3-DAY13-LeTrungKien-02748-Monitoring-LLMOps
- **Commit SHA cuối:** (Sẽ điền SHA sau khi git commit)
- **Challenge ID:** day13-k4-l3b-monitoring-llmops-v1
- **Tên project Langfuse cá nhân:** `day13-k4-l3b-02748`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 30/100 | 100/100 | Đã hoàn thành schema, correlation ID, enrichment và PII scrubbing |
| `validate_dashboard.py` | 6/6 panel | 6/6 panel | Đã thiết lập 6 panel chuẩn và đúng contract |
| `pytest` | 22 passed | 22 passed | Passed toàn bộ 22/22 unit tests |
| Số traces hợp lệ | | | |
| Số PII leak | | 0 | Đã che hoàn toàn các thông tin PII mẫu |
| Latency P95 / TTFT P95 | | | |
| Retrieval success rate | | | |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Middleware `CorrelationIdMiddleware` kiểm tra header `x-request-id` từ client, nếu không có sẽ khởi tạo ID theo định dạng `req-<8-hex>`. Mã này được gán vào `structlog` contextvars qua `bind_contextvars(correlation_id=...)` và trả về trong response header `x-request-id` cũng như `x-response-time-ms`.
- **Các metadata được ghi vào structured log:** `user_id_hash` (mã SHA-256 từ user_id), `session_id`, `feature`, `model`, `env`, `ts`, `level`, `correlation_id`.
- **Cách bảo đảm PII được scrub trước khi ghi:** Đăng ký processor `scrub_event` trong cấu hình `structlog.configure` chạy trước khi render JSON và ghi file `data/logs.jsonl`. Processor này sử dụng các regex pattern định sẵn trong `app/pii.py` để thay thế thông tin nhạy cảm thành `[REDACTED_*]`.
- **Cách kiểm chứng kết quả:** Xóa file log cũ, khởi động lại API, chạy `python scripts/load_test.py` và kiểm tra lại bằng `python scripts/validate_logs.py` đạt điểm 100/100.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Truy cập Langfuse Cloud trong project `day13-k4-l3b-02748` tạo bằng API keys cá nhân.
- **Cấu trúc root/retrieval/generation observations:** Root observation `lab-agent-run` (as_type="agent"), chứa 2 child observations: `retrieval` (as_type="retriever") và `generation` (as_type="generation") ghi nhận đủ model, prompt, usage và cost.
- **Cách nối trace với log:** Sử dụng trường `correlation_id` được ghi thống nhất vào cả `structlog` contextvars và `Langfuse trace metadata`.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** Version 1 (`baseline`, `production`)
- **Version/label candidate:** Version 2 (`candidate`)
- **Trace ID của mỗi version:** (Sẽ điền sau khi chạy workload thực tế)
- **Cách promote và rollback `production`:** Chuyển nhãn `production` từ Version 1 sang Version 2 trên Langfuse UI để Promote. Khi cần Rollback, chuyển nhãn `production` quay lại trỏ về Version 1 mà không cần thay đổi hay restart code ứng dụng.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** 6 panel chuẩn gồm Latency (P50/P95/P99 + TTFT P95), Traffic (Count, request/phút), Errors (Error rate, breakdown, retrieval success), Cost (USD), Tokens (In/Out), Quality (Mean quality score).
- **SLO và lý do chọn:** Fast successful requests: Target 99.5% requests thành công và latency <= 3000ms trong cửa sổ 28 ngày. Lý do chọn nhằm đảm bảo trải nghiệm phản hồi nhanh cho người dùng ứng dụng LLM.
- **Cách tính error budget:** Với SLO 99.5% trong 28 ngày, error budget là 0.5%. Nếu trong 28 ngày có 10,000 request thì hệ thống được phép có tối đa 50 request bị chậm (>3000ms) hoặc bị lỗi (500).
- **Ba alert và runbook tương ứng:**
  1. `HighLatencyP95` (Warning, p95 > 3000ms trong 5m) -> Runbook [docs/alerts.md#alert-1](file:///c:/Users/kient/Vin_projects/day_13/K4-L3-DAY13-LeTrungKien-02748-Monitoring-LLMOps/docs/alerts.md#alert-1)
  2. `HighErrorRate` (Critical, error rate > 5% trong 2m) -> Runbook [docs/alerts.md#alert-2](file:///c:/Users/kient/Vin_projects/day_13/K4-L3-DAY13-LeTrungKien-02748-Monitoring-LLMOps/docs/alerts.md#alert-2)
  3. `RetrievalFailureRate` (Warning, retrieval success < 90% trong 3m) -> Runbook [docs/alerts.md#alert-3](file:///c:/Users/kient/Vin_projects/day_13/K4-L3-DAY13-LeTrungKien-02748-Monitoring-LLMOps/docs/alerts.md#alert-3)

## 7. Điều tra challenge

- **Challenge ID:** `day13-k4-l3b-monitoring-llmops-v1`
- **Khoảng thời gian điều tra:** 2026-09-30 09:50 - 10:40 (Asia/Ho_Chi_Minh)
- **Triệu chứng từ metrics:** Dashboard Latency tăng vọt, chỉ số P95 latency vượt ngưỡng `2000ms` (đạt mức ~2900ms - 3000ms), trong khi Traffic vẫn duy trì mức bình thường.
- **Log line và correlation ID liên quan:** Log event `response_sent` thu được có `correlation_id` thực tế là `req-366e8e9b` (cùng các request `req-7ccb7bd6`, `req-49226d05`, `req-fd270fcb`, `req-a54bdfe6`).
- **Trace ID và span gây ảnh hưởng:** Mở Trace tương ứng trên Langfuse có cùng `correlation_id` (`req-366e8e9b`), quan sát thấy root observation `lab-agent-run` bị kéo dài do child span `retrieval` phát sinh độ trễ lớn (delay 2.5s trong `retrieve()`).
- **Root cause:** Incident `rag_slow` gây ra độ trễ lớn (delay 2.5s) trong tầng truy vấn tài liệu `retrieve()`.
- **Fix action:** Tắt sự cố bằng `python scripts/inject_incident.py --disable` hoặc khôi phục dịch vụ vector store / RAG corpus.
- **Preventive measure:** Áp dụng Alert rule `HighLatencyP95` để phát hiện sớm trễ và thiết lập timeout ngắn hơn cho bước `retrieve()` kết hợp fallback cache.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đăng ký PII Processor (`scrub_event`) trong `structlog` trước bước render JSON và ghi file để đảm bảo mọi thông tin nhạy cảm (Email, Phone VN, CCCD, Thẻ) được scrub hoàn toàn tự động ở cấp độ logger.
- **Một lỗi/blocker đã gặp:** Khi bổ sung `update_current_generation`, mock client `RecordingLangfuseClient` trong unit test thiếu phương thức này dẫn tới quăng ngoại lệ `AttributeError`.
- **Cách tìm nguyên nhân và xử lý:** Đọc kĩ traceback trong pytest output, kiểm tra cấu trúc của `RecordingLangfuseClient` và cập nhật kiểm tra an toàn `hasattr` + `callable` trước khi gọi.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics phát hiện triệu chứng hệ thống bị xấu (P95 latency tăng) -> Logs dựa vào khoảng thời gian đó để khoanh vùng và lấy `correlation_id` -> Traces dùng `correlation_id` để soi waterfall span tree và chỉ ra bước `retrieval` bị chậm.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Quản lý prompt độc lập giúp thử nghiệm phiên bản mới an toàn, có thể rollback tức thì qua nhãn `production` khi xảy ra regression mà không cần sửa code. Theo dõi token/cost giúp ngăn ngừa bùng nổ chi phí (cost spike).
- **Điều quan trọng nhất đã học:** Cách thiết lập hệ thống observability chuẩn hóa cho AI API để chuyển ứng dụng từ "hộp đen" thành hệ thống có thể điều tra vết lỗi chính xác.
- **Hạn chế còn lại, nếu có:** Cần chụp lại các hình ảnh screenshot đính kèm thực tế vào thư mục `submission/evidence/` để hoàn thiện 100% tài liệu nộp bài.

## 9. Checklist trước khi nộp

- [ ] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [ ] Repository chạy lại được theo README.
- [ ] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
