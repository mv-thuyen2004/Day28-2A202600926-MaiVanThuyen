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
   - **Trade-off (Chi phí & Tài nguyên)**: Tách biệt GPU lên Kaggle giúp tiết kiệm chi phí phần cứng local (Local CPU/Memory nhẹ nhàng) nhưng đánh đổi bằng độ trễ mạng (Network Latency) lớn hơn do kết nối qua ngrok tunnel.
   - **Performance**: Tối ưu hóa bằng cách chạy mô hình LLM dạng quantized (Int4 GPTQ) trên vLLM để tăng tốc độ sinh từ (tokens/sec) và giảm dung lượng VRAM.
   - **Reliability**: Sử dụng kiến trúc Event-driven qua Kafka và Prefect giúp đảm bảo dữ liệu thô (raw data) không bị mất mát khi lưu trữ trung gian qua Delta Lake.
   - **Maintainability**: Cấu hình tách biệt hoàn toàn qua Docker Compose giúp dễ quản lý cấu hình các service, dễ dàng tích hợp và mở rộng độc lập.

2. **Trong kiến trúc hybrid (Local + Kaggle), bạn xử lý ngắt kết nối giữa local và Kaggle như thế nào? Có cơ chế fallback không?**
   - **Xử lý ngắt kết nối**: Trong API Gateway, cơ chế HTTP client sử dụng `timeout` giới hạn (ví dụ: 30s) và bắt các Exception kết nối (`httpx.ConnectError`, `httpx.TimeoutException`).
   - **Cơ chế Fallback**:
     - *Với Embedding*: Nếu dịch vụ embedding trên Kaggle bị lỗi, hệ thống có thể chuyển sang sử dụng mô hình local CPU nhỏ hoặc trả về vector mặc định để tránh crash hệ thống.
     - *Với LLM*: Nếu vLLM bị mất kết nối, API Gateway sẽ trả về thông báo lỗi thân thiện cho client thay vì làm sập toàn bộ luồng, hoặc thiết lập route dự phòng gọi sang các API thương mại (như OpenAI/Gemini API) làm phương án dự phòng (fallback API).

3. **Giải thích cách event-driven architecture với Kafka giúp decouple các components trong AI platform của bạn.**
   - **Decouple Ingestion & Processing**: Client đẩy dữ liệu vào topic `data.raw` của Kafka mà không cần biết khi nào dữ liệu được xử lý.
   - **Khả năng chịu tải**: Nếu Prefect pipeline hoặc Delta Lake bị quá tải hay dừng đột ngột, Kafka vẫn lưu trữ và giữ hàng đợi tin nhắn an toàn (Message Persistence). Khi Prefect worker hoạt động trở lại, nó sẽ tiếp tục tiêu thụ dữ liệu từ offset cũ mà không làm thất thoát thông tin.

4. **Bạn đã implement observability như thế nào? Logs, metrics, và traces được thu thập và visualized ra sao?**
   - **Logs**: Thu thập trực tiếp từ log của docker container cho từng service thông qua cơ chế Docker logging.
   - **Metrics**: API Gateway tích hợp `prometheus-fastapi-instrumentator` để expose endpoint `/metrics`. Prometheus định kỳ scrape các metrics này và Grafana kết nối vào Prometheus để trực quan hóa đồ thị tải, lượng request, và độ trễ.
   - **Traces**: Sử dụng LangSmith để theo dõi chi tiết luồng xử lý (LLM tracing, input/output prompt, latency của từng bước trong chain).

5. **Nếu một service trong stack (ví dụ: Qdrant hoặc Kafka) bị crash, hệ thống của bạn sẽ xử lý như thế nào? Có graceful degradation không?**
   - **Qdrant crash**: API Gateway bắt lỗi khi tìm kiếm vector. Thay vì trả về lỗi 500, hệ thống sẽ thực hiện suy diễn LLM không có ngữ cảnh bổ sung (No RAG context) hoặc cảnh báo cho người dùng (Graceful Degradation).
   - **Kafka crash**: Script ingestion sẽ kích hoạt cơ chế retry cục bộ hoặc ghi tạm dữ liệu thô ra file log local để đẩy lại sau khi Kafka hồi phục.
   - **Redis (Feature Store) crash**: API Gateway có thể bỏ qua bước làm giàu dữ liệu (feature enrichment) từ Feast và sử dụng thông tin thô từ query để tiếp tục xử lý.

## Câu Hỏi Thêm?
Liên hệ giảng viên qua LMS hoặc office hours.

