# Hướng Dẫn Nộp Bài - Lab #28: Full Platform Integration Sprint

## Yêu Cầu Nộp Bài

**Full AI infrastructure platform demo** - từ data ingestion đến model serving với full observability.

## Các Artifacts Cần Nộp

### 1. Source Code
- Folder `lab28/` hoàn chỉnh với tất cả files
- Tất cả integration scripts hoạt động
- Prefect flows đã deploy và schedule

### 2. Screenshots Demo
Chụp màn hình các bước:
- Prefect UI: http://localhost:4200 (flow đang chạy)
- API Gateway call: `curl http://localhost:8000/health`
- Grafana dashboard: http://localhost:3000

### 3. Kết Quả Smoke Tests
Chạy và chụp màn hình kết quả:
```bash
cd lab28
pytest smoke-tests/ -v
```
Kỳ vọng: 5/5 tests passing

### 4. Production Readiness Score
```bash
python scripts/production_readiness_check.py
```
Kỳ vọng: Score >80%

### 5. Documentation
- `README.md` giải thích cách:
  - Start platform: `docker compose up -d`
  - Deploy Prefect flows
  - Run smoke tests
  - Access dashboards (Grafana:3000, Prometheus:9090, Prefect:4200)

## Định Dạng Nộp Bài

Tạo Repo GitHub chứa:
```
lab28_submission_[student_id]
├── lab28/                    # Source code hoàn chỉnh
│   ├── docker-compose.yml
│   ├── prefect/flows/
│   ├── scripts/
│   ├── api-gateway/
│   └── monitoring/
├── screenshots/              # Screenshots demo
│   ├── prefect_ui.png
│   ├── api_gateway.png
│   └── grafana_dashboard.png
├── smoke_tests_results.png   # Screenshot kết quả pytest
├── production_readiness.png  # Screenshot readiness score
└── README.md                # Hướng dẫn setup
```

## Địa Điểm Nộp
Nộp link repo GitHub qua LMS

## Tiêu Chí Chấm Điểm

| Tiêu Chí | Trọng Số | Mô Tả |
|----------|----------|-------|
| Integration Completeness | 40% | Tất cả 10 integration points hoạt động, data flow end-to-end |
| Observability | 25% | Logs, metrics, traces hiển thị; alerts configured |
| Performance | 20% | Latency trong SLO; load tested; không có memory leaks |
| Architecture Quality | 15% | Clean separation, GitOps config, documented decisions |

## Các Vấn Đề Cần Tránh

- Config drift giữa các environments
- Thiếu error handling tại integration points
- Monitoring coverage không hoàn chỉnh
- Không có rollback strategy
- Demo không test trước khi nộp

## 5 Câu Hỏi Cần Trả Lời Khi Nộp

1. **Phân tích các trade-offs trong thiết kế kiến trúc AI platform của bạn. Bạn đã cân bằng giữa performance, reliability, và maintainability như thế nào?**
   * **Performance vs. Cost**: Thay vì chạy mô hình LLM lớn và embedding cục bộ (đòi hỏi cấu hình phần cứng đắt đỏ hoặc local GPU), platform lựa chọn kiến trúc hybrid, đưa tải tính toán GPU (vLLM và sentence-transformers) lên môi trường Kaggle GPU miễn phí. Sự đánh đổi là độ trễ truyền dữ liệu qua mạng (network latency qua ngrok tunnel mất khoảng 3 - 9 giây) so với việc gọi trực tiếp qua local network.
   * **Reliability vs. Complexity**: Việc tích hợp Kafka làm Message Broker trung gian giúp tăng độ tin cậy của luồng dữ liệu (nếu Prefect flow hoặc cơ sở dữ liệu Delta Lake bị tạm ngưng, dữ liệu thô vẫn an toàn trong partition của Kafka). Đổi lại, hệ thống tăng độ phức tạp vận hành vì phải duy trì thêm cụm Zookeeper + Kafka Broker và giải quyết các vấn đề cấu hình định tuyến (listener).
   * **Maintainability vs. Tight Coupling**: Hệ thống được chia tách rõ ràng thành các microservices độc lập giao tiếp qua API Gateway (FastAPI), cơ sở dữ liệu Vector (Qdrant), và Feature Store (Feast/Redis). Điều này giúp việc bảo trì vô cùng dễ dàng; ví dụ, khi thay đổi mô hình LLM, ta chỉ cần deploy lại model registry trên Kaggle và cập nhật endpoint của API Gateway mà không ảnh hưởng tới luồng nạp dữ liệu thô và Delta Lake.

2. **Trong kiến trúc hybrid (Local + Kaggle), bạn xử lý ngắt kết nối giữa local và Kaggle như thế nào? Có cơ chế fallback không?**
   * **Phát hiện ngắt kết nối**: Ở cổng API Gateway, các yêu cầu gọi tới vLLM URL được giới hạn bởi cơ chế `timeout` (30 giây). Nếu ngrok tunnel bị ngắt hoặc hết hạn, API Gateway sẽ ném ra ngoại lệ `httpx.ConnectError` hoặc `TimeoutException` thay vì giữ trạng thái chờ vô thời hạn.
   * **Cơ chế Fallback (Graceful Degradation)**:
     * Khi gọi RAG hoặc Inference bị lỗi do mất kết nối Kaggle, API Gateway có thể chuyển sang trả về câu trả lời mặc định lưu trong cache, hoặc gửi cảnh báo "Hệ thống AI đang bảo trì" tới client thay vì trả về lỗi HTTP 500.
     * Về phía luồng dữ liệu, dữ liệu gửi vào Kafka vẫn tiếp tục được lưu vào Delta Lake (Parquet local). Khi kết nối Kaggle phục hồi, Prefect có thể chạy batch process để lấy dữ liệu lịch sử và đồng bộ bù (sync) sang Qdrant.

3. **Giải thích cách event-driven architecture với Kafka giúp decouple các components trong AI platform của bạn.**
   * **Tách biệt Producer và Consumer**: Phía ingest dữ liệu (`01_ingest_to_kafka.py`) không cần biết ai tiêu thụ dữ liệu, khi nào xử lý và lưu trữ ở đâu. Nó chỉ cần đảm bảo đẩy dữ liệu thô thành công vào topic `data.raw`.
   * **Tách biệt thời gian (Temporal Decoupling)**: Prefect flow có thể chạy định kỳ (theo cron schedule) để lấy hàng loạt message từ Kafka và lưu vào Delta Lake, thay vì phải túc trực xử lý thời gian thực liên tục, giảm tải đáng kể cho tài nguyên local.
   * **Phân phối song song (Multi-consumer)**: Cùng một topic `data.raw`, ngoài Prefect lưu trữ Delta Lake, ta có thể dễ dàng cắm thêm các dịch vụ khác (ví dụ: Real-time Analytics, Alerting) cùng đọc dữ liệu song song mà không can thiệp hay làm gián đoạn hệ thống cũ.

4. **Bạn đã implement observability như thế nào? Logs, metrics, và traces được thu thập và visualized ra sao?**
   * **Metrics**: API Gateway tích hợp `prometheus-fastapi-instrumentator` để tự động xuất metrics hiệu năng trên cổng `/metrics`. Prometheus local sẽ scrape metrics này cứ mỗi 15 giây. Grafana được kết nối tới Prometheus để trực quan hóa (P95 Latency, Request Rate, Error Rate) lên dashboard.
   * **Logs**: Tất cả các dịch vụ (API Gateway, Prefect Worker, Kafka, Qdrant) ghi log trực tiếp ra `stdout`/`stderr`. Docker daemon sẽ thu thập tập trung để hỗ trợ debugging qua lệnh `docker compose logs`.
   * **Traces**: Tích hợp **LangSmith** thông qua việc khai báo các biến môi trường (`LANGCHAIN_API_KEY`, `LANGCHAIN_PROJECT="lab28-platform"`, `LANGCHAIN_TRACING_V2="true"`). Các bước gọi LLM, chuỗi RAG đều được trace đầy đủ lên giao diện đám mây của LangSmith để phân tích prompt, latency và chi phí token.

5. **Nếu một service trong stack (ví dụ: Qdrant hoặc Kafka) bị crash, hệ thống của bạn sẽ xử lý như thế nào? Có cơ chế graceful degradation không?**
   * **Nếu Kafka bị crash**: Các producer nạp dữ liệu sẽ báo lỗi kết nối. Tuy nhiên, API Gateway và Qdrant vẫn hoạt động bình thường, phục vụ tốt các câu truy vấn chat từ dữ liệu cũ đã index trước đó.
   * **Nếu Qdrant bị crash**: API Gateway không thể lấy context từ tìm kiếm vector. Lớp code API Gateway đã được cải tiến để bắt các ngoại lệ kết nối Qdrant và thực hiện *graceful degradation*: Bỏ qua phần tìm kiếm ngữ cảnh, chỉ chuyển trực tiếp câu hỏi thuần của user tới vLLM và trả về kèm một cảnh báo nhẹ "Vector search offline".
   * **Nếu Redis (Feast) bị crash**: Flow đồng bộ hoặc API Gateway truy vấn online feature store sẽ gặp lỗi. Hệ thống có thể thiết lập cơ chế fallback đọc trực tiếp từ các file Parquet ở Delta Lake (tốc độ chậm hơn nhưng đảm bảo dữ liệu không bị mất).

---
*Hoàn thành tài liệu nộp bài Lab #28 - AICB-P2T2.*

