# Báo cáo Lab 28 — Modern AI Platform Integration

## Thông tin bài làm

- Họ và tên: Phạm Công Đạt
- Mã học viên: 2A202601406
- Hình thức: cá nhân
- Nhánh nộp bài: `main` (bài cá nhân, nộp trực tiếp)
- Sơ đồ kiến trúc: [docs/images/lab28-architecture-overview.png](docs/images/lab28-architecture-overview.png)

## Phần mã đã hoàn thiện

Bài làm chỉ sửa bốn hàm được yêu cầu trong
`src/lab28_platform/integration_tasks.py`:

1. `event_headers`: luôn truyền `idempotency-key` dưới dạng bytes; chỉ truyền
   `traceparent` khi có trace hợp lệ, không phát chuỗi rỗng.
2. `dedupe_latest`: đọc iterable đúng một lần, giữ một sự kiện cho mỗi
   `idempotency_key`, chọn bản mới nhất bằng cặp `(occurred_at, event_id)` và
   sắp xếp đầu ra theo khóa để kết quả không phụ thuộc thứ tự Kafka giao bản tin.
3. `feast_online_request`: dùng danh sách `FEATURE_REFS` chuẩn trong
   `contracts.py`, entity `asker_id` và `full_feature_names=false`.
4. `readiness_status`: lỗi bắt buộc trả `not_ready`; chỉ lỗi tùy chọn trả
   `degraded`; không có lỗi trả `ready`.

## Kết quả xác minh

| Hạng mục | Kết quả |
| --- | --- |
| Starter tests và unit tests | 87 passed |
| Ruff | Đạt, không có lỗi |
| Integration matrix | 245 checks passed |
| Portability | Đạt |
| Kubernetes/GitOps manifests | Đạt |
| Docker Compose core và full config | Hợp lệ |
| Core stack | Các container bắt buộc running/healthy |
| Qdrant | 20 points, trạng thái `ready` |
| MLflow | `lab28-rag-release`, champion version 2 |
| Seed qua Envoy gateway | 3 documents và 3 feedback được chấp nhận |
| J1 golden path | 12 passed, 3 GPU-gated skipped |
| J2 idempotent replay | 9 passed |
| Full integration không GPU/LangSmith | 56 passed, 16 deselected |
| Integration report | Score 83; 5/6 điểm tự probe đạt |
| vLLM thật | UNVERIFIED — không có endpoint GPU do lớp cung cấp |
| LangSmith | UNVERIFIED — không có credential do lớp cung cấp |

Hai gate vLLM và LangSmith không được giả lập. Core profile vì vậy có thể ở
`degraded` khi vLLM là thành phần tùy chọn; nếu cấu hình bắt buộc vLLM thật thì
trạng thái đúng là `not_ready`.

## Luồng đúng, replay và phiên bản

Luồng được kiểm tra theo thứ tự Envoy → FastAPI → Kafka → Airflow → Delta →
Feast/Qdrant → API. Request ID và W3C trace ID được truyền qua header; Kafka
đồng thời giữ idempotency key. Delta MERGE chỉ giữ bản mới nhất theo khóa nên
gửi lại cùng dữ liệu không tạo thêm bản ghi. Qdrant dùng point ID ổn định và
MLflow dùng alias `champion`, vì vậy phiên bản phục vụ có thể đổi hoặc rollback
mà không sửa mã nguồn.

Phiên bản release cục bộ dùng để kiểm tra:

- MLflow run ID: `8d5fb83b796d4630be6e7b0ea89e408d`
- MLflow model version/alias: `2` / `champion`
- Prompt version: `v1`
- Embedding model: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2@faf4aa4225822f3bc6376869cb1164e8e3feedd0`

Các trace ID, DAG run ID, Delta version, metric snapshot và bằng chứng phục hồi
được sinh trực tiếp vào thư mục `evidence/` bởi live integration tests. Thư mục
này được repo chủ động ignore để không commit dữ liệu chạy cục bộ.

Evidence cuối ghi nhận DAG run `it-8adc8b7c` thành công với cả bốn task; Delta
documents ở version 9 với 10 hàng, feedback ở version 15 với 16 hàng. Prometheus
thấy chín target bắt buộc `up`; target vLLM tùy chọn `down` đúng với trạng thái
UNVERIFIED. Bundle có đủ các file IP01–IP10 cùng bằng chứng Grafana và
`integration-report.json`.

Trong lần kích hoạt Airflow đầu tiên ngay sau cold-start, consumer group chưa
hoàn tất rebalance trong ba poll nên batch trả `polled=0`. Kafka chưa commit
offset; lần chạy kế tiếp xử lý toàn bộ backlog và J1 đạt. Sự cố này chứng minh
dữ liệu không mất, đồng thời cho thấy production cần warm-up/assignment wait và
metric cảnh báo empty batch sau khi vừa có ingestion.

## Kết quả tải

Baseline gọi `/ready` qua Envoy; máy kiểm tra có 12 logical CPU, Python 3.11.9,
Docker Desktop Linux và không có GPU/vLLM thật.

| Requests | Workers | HTTP 200 | Client ghi lỗi | P50 | P95 | P99 |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 200 | 8 | 39 | 161 | 4.89 ms | 595.93 ms | 1451.56 ms |
| 200 | 16 | 13 | 187 | 6.76 ms | 729.25 ms | 1062.08 ms |

Script chuẩn dùng `urllib` và gom mọi `HTTPError` thành status `0`; đối chiếu
evidence gateway cho thấy phần lớn lỗi này là HTTP 429 do policy 10 RPS, không
phải API mất kết nối. Khi tăng concurrency, số request được nhận giảm rõ rệt;
bottleneck quan sát được là rate limiter ở edge. Các số liệu laptop này chỉ là
baseline, không được dùng để suy ra capacity production. Chưa đo `/api/v1/ask`
vì gate vLLM thật không được cấp.

## Quyết định và đánh đổi kỹ thuật

- Idempotency tại biên kết hợp với Delta MERGE cho cơ chế at-least-once dễ vận
  hành hơn exactly-once xuyên toàn hệ thống. Đổi lại, mọi sink phải có khóa ổn
  định và quy tắc chọn bản mới nhất rõ ràng.
- Sắp xếp kết quả dedupe làm kết quả có tính xác định và dễ kiểm thử, với chi phí
  `O(k log k)` cho `k` khóa duy nhất. Chi phí này phù hợp cho batch của lab.
- Dùng registry contract của Feast thay vì lặp lại tên feature tránh drift giữa
  producer và online serving, nhưng thay đổi schema cần quy trình version/migrate.
- Alias `champion` của MLflow tách deployment khỏi model version cụ thể và hỗ
  trợ rollback nhanh; cần audit và quyền phê duyệt alias trong production.
- Readiness phân biệt dependency bắt buộc và tùy chọn để hệ thống còn phục vụ có
  kiểm soát khi feature store/vector store/vLLM lỗi. Response phải công khai
  `degraded` và lý do để không che giấu chất lượng giảm.
- Tracing cục bộ qua OpenTelemetry/Jaeger là bằng chứng xác định khi offline;
  LangSmith chỉ được công nhận khi có credential thật.

## Khoảng trống trước production

- Bổ sung TLS/mTLS, xác thực, RBAC, secret manager và xoay vòng credential cho
  Kafka, Airflow, MLflow, Feast, Qdrant cùng các dashboard.
- Thay volume cục bộ bằng object storage/database có backup, mã hóa, retention,
  kiểm thử restore và kế hoạch disaster recovery.
- Triển khai Kafka, Airflow, MLflow và Qdrant theo mô hình HA; đặt quota,
  autoscaling, pod disruption budget và capacity plan.
- Bổ sung GPU endpoint vLLM thật, kiểm soát model provenance, giới hạn token,
  batching, autoscaling, timeout/circuit breaker và kiểm thử tải dài hạn.
- Thiết lập SLI/SLO, alert routing, error budget; kiểm soát PII và sampling/retention
  cho log, metric, trace.
- Áp dụng GitOps trên cluster thật với signed image/SBOM, admission policy,
  canary, drift detection và diễn tập rollback định kỳ. Bài lab mới xác minh
  contract manifest tĩnh, chưa chứng minh rollout trên cluster production.

## Đóng góp cá nhân

Vì đây là bài cá nhân, Phạm Công Đạt thực hiện toàn bộ vai trò: đọc contract và
ma trận IP01–IP10, hoàn thiện bốn hàm cốt lõi, chạy kiểm thử tĩnh/live, kiểm tra
Docker và manifest, thu bằng chứng, phân tích trade-off, chuẩn bị demo và nộp
trực tiếp trên nhánh `main`. Không có thành viên khác tham gia.

## Trình tự demo đề xuất

1. Mở sơ đồ kiến trúc và giải thích ba vùng ingestion, data/ML và serving.
2. Gửi một request qua gateway, theo request/trace ID tới Kafka, Airflow, Delta,
   Feast/Qdrant và đối chiếu MLflow champion.
3. Gửi lại cùng idempotency key và chứng minh số hàng Delta/point Qdrant không
   tăng ngoài dự kiến.
4. Dừng một dependency tùy chọn, quan sát `degraded`, metric/trace, sau đó khôi
   phục và chứng minh hệ thống trở lại `ready` mà không mất dữ liệu.
5. Trình bày P50/P95/P99, điểm nghẽn quan sát được, một đánh đổi và các khoảng
   trống cần xử lý trước production.
