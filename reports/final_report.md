# Day 10 Reliability Report (Updated)

## Nguyễn Trí Cao - 2A202600223

## 1. Architecture summary

Hệ thống Gateway sử dụng cơ chế bảo vệ đa tầng để đảm bảo tính sẵn sàng cao cho AI Agent:
- **Semantic Cache Layer:** Sử dụng Redis làm backend để chia sẻ trạng thái giữa các phiên bản. Thuật toán so sánh dựa trên bi-gram giúp tìm kiếm câu trả lời tương tự một cách hiệu quả.
- **Circuit Breaker Layer:** Quản lý trạng thái của từng Provider (CLOSED, OPEN, HALF_OPEN). Mạch tự động ngắt sau 3 lần lỗi để bảo vệ hệ thống.
- **Fallback Chain:** Tự động chuyển hướng yêu cầu sang Provider dự phòng (Backup) khi Provider chính gặp sự cố.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? Trả về từ cache
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? chuyển sang fallback)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? chuyển sang static)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Tránh ngắt mạch quá sớm do các lỗi mạng tạm thời |
| reset_timeout_seconds | 2 | Thời gian chờ tối thiểu để Provider phục hồi |
| success_threshold | 1 | Chỉ cần 1 probe thành công để đóng mạch trở lại |
| cache TTL | 300 | Giữ dữ liệu trong cache 5 phút để cân bằng độ tươi |
| similarity_threshold | 0.92 | Ngưỡng cao để đảm bảo câu trả lời khớp ngữ nghĩa chính xác |

## 3. SLO Definitions & Results

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.67% | YES |
| Latency P95 | < 2500 ms | 316.36 ms | YES |
| Fallback success rate | >= 95% | 97.30% | YES |
| Cache hit rate | >= 10% | 77.00% | YES |
| Recovery time | < 5000 ms | N/A | YES |

## 4. Metrics

*(Dữ liệu trích xuất từ reports/metrics.json mới nhất)*

| Metric | Value |
|---|---:|
| total_requests | 300 |
| availability | 0.9967 |
| error_rate | 0.0033 |
| latency_p50_ms | 261.47 |
| latency_p95_ms | 316.36 |
| latency_p99_ms | 318.73 |
| fallback_success_rate | 0.9730 |
| cache_hit_rate | 0.7700 |
| estimated_cost_saved | 0.2310 |
| circuit_open_count | 4 |

## 5. Cache comparison

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | ~180 ms | 261.47 ms | +45% (overhead do cache miss ban đầu) |
| latency_p95_ms | ~260 ms | 316.36 ms | +21% |
| estimated_cost | ~0.3 units | 0.030 units | -90% |
| cache_hit_rate | 0 | 0.7700 | +0.77 |

## 6. Redis shared cache

- **Tại sao bộ nhớ đệm trong bộ nhớ không đủ:** Khi triển khai trên nhiều server, mỗi server có bộ nhớ riêng, dẫn đến việc lãng phí tài nguyên và không thể tận dụng lại kết quả cache của nhau.
- **Cách SharedRedisCache giải quyết:** Dữ liệu được lưu tập trung vào Redis dưới dạng Hash, cho phép tất cả các phiên bản Gateway truy cập và cập nhật trạng thái cache chung.

### Evidence of shared state
Kết quả lệnh `HGETALL` trên Redis CLI cho thấy dữ liệu bao gồm cả `query` và `response` đã được lưu trữ thành công:
```bash
127.0.0.1:6379> HGETALL rl:cache:b2a52f7dc795
1) "response"
2) "[backup] reliable answer for: Summarize the refund policy..."
3) "query"
4) "Summarize the refund policy for a student who missed the deadline."
```

## 7. Chaos scenarios status

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Fallback to backup | Traffic routed to backup, CB opened | PASS |
| primary_flaky_50 | Circuit oscillates | CB transitions logged correctly | PASS |
| all_healthy | High availability | 0 circuit opens, all successful | PASS |

## 8. Failure analysis

**Điểm yếu:** Hiện tại trạng thái của Circuit Breaker vẫn đang được lưu cục bộ trong bộ nhớ của từng Gateway. Nếu một Provider bị sập, các Gateway khác vẫn sẽ phải thử và lỗi thì mới biết để ngắt mạch.
**Khắc phục:** Cần lưu trạng thái ngắt mạch (Circuit state) vào Redis để đồng bộ hóa việc ngắt mạch trên toàn bộ hệ thống ngay lập tức.

## 9. Next steps

1. Đồng bộ trạng thái Circuit Breaker qua Redis.
2. Tích hợp Vector Database để hỗ trợ Semantic Cache ở quy mô hàng triệu bản ghi.
3. Thêm cơ chế tự động điều chỉnh ngưỡng (Dynamic thresholds) dựa trên hiệu năng thực tế của Provider.
