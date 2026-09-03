# Evidence bundle — Lab 28

Người thực hiện: **Phạm Công Đạt — 2A202601406**

Nhánh nộp bài: **`main`**

## Nội dung

- `integration-report.json`: báo cáo tự động của integration matrix.
- `ip01-*.json` đến `ip10-*.json`: bằng chứng live cho mười điểm tích hợp;
  IP09 có hai file Prometheus và Grafana theo contract.
- `fast-suite.txt`: output starter/unit tests.
- `integration-suite.txt`: output full integration suite không GPU/LangSmith.
- `j3-provenance.txt`: kết quả promotion/rollback và provenance của MLflow.
- `ruff-check.txt`, `matrix-check.txt`, `portability-check.txt`,
  `manifest-check.txt`: kết quả các kiểm tra tĩnh bắt buộc.
- `load-profile.json`: P50/P95/P99 ở 8 và 16 workers cùng diễn giải rate limit.

## Gate môi trường

Không có endpoint vLLM GPU thật hoặc credential LangSmith do lớp cung cấp. IP07
và LangSmith được ghi `UNVERIFIED`/`not_ready` thay vì giả lập, đúng hướng dẫn
trong `SUBMISSION.md`.

J3 phát hành tạm model version 6 từ champion version 2, ghi đầy đủ provenance
vào `ip06-mlflow-release.json`, rồi rollback alias về version 2. Vì thế
`integration-report.json` ghi champion hiện tại là version 2 trong khi evidence
J3 ghi version 6 đã được promotion trong bài kiểm thử rollback.

Bundle không chứa token, mật khẩu, `.env`, database, cache, model weights hoặc
thư mục `.lab28/`.
