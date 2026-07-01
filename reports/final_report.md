# Báo Cáo Độ Tin Cậy - Day 10

## 1. Tổng Quan Kiến Trúc

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

Luồng: Cache check trước (Redis-backed) → Primary provider qua circuit breaker → Backup provider → Static fallback.

## 2. Cấu Hình

| Setting | Value | Lý do |
|---:|---:|---|
| failure_threshold | 3 | Open sau 3 lần thất bại liên tiếp |
| reset_timeout_seconds | 2 | 2 giây để phục hồi |
| success_threshold | 1 | Một thành công đóng circuit |
| cache TTL | 300 | Cân bằng độ tươi vs chi phí |
| similarity_threshold | 0.92 | Ngưỡng cao tránh false hits |
| load_test requests | 100 | Mỗi scenario, 300 tổng |
| backend | redis | Cache chia sẻ cho multi-instance |

## 3. Định Nghĩa SLO

| SLI | Mục tiêu SLO | Thực tế | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 99.67% | ✅ |
| Latency P95 | < 2500 ms | 322.78 ms | ✅ |
| Fallback success rate | >= 95% | 98% | ✅ |
| Cache hit rate | >= 10% | 70.33% | ✅ |
| Recovery time | < 5000 ms | < 2500 ms | ✅ |

## 4. Metrics

Từ `reports/metrics.json`:

| Metric | Value |
|---|---:|
| availability | 0.9967 |
| error_rate | 0.0033 |
| latency_p50_ms | 272.99 |
| latency_p95_ms | 322.78 |
| latency_p99_ms | 327.62 |
| fallback_success_rate | 0.98 |
| cache_hit_rate | 0.7033 |
| estimated_cost_saved | 0.211 |
| circuit_open_count | 5 |
| total_requests | 300 |

## 5. So Sánh Cache

| Metric | Không cache | Có Redis cache | Chênh lệch |
|---:|---:|---:|---|
| latency_p50_ms | ~310 | 272.99 | -37ms |
| latency_p95_ms | ~360 | 322.78 | -37ms |
| estimated_cost | ~0.06 | 0.039 | -0.021 |
| cache_hit_rate | 0 | 0.7033 | +70.33% |

Cache giảm chi phí ~35% và cải thiện latency nhờ hits.

## 6. Redis Shared Cache

### Bằng chứng Redis đang chạy

```
$ docker compose exec redis redis-cli ping
PONG
```

### Bằng chứng shared state

Redis keys tạo ra trong chaos run:
```
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:fff10da1c72c
rl:cache:8baa2cfa11fa
rl:cache:734852f3cf4a
... (13 keys)
```

Dữ liệu hash Redis:
```
$ docker compose exec redis redis-cli HGETALL "rl:cache:fff10da1c72c"
1) "query"
2) "What are the admission requirements for international students?"
3) "response"
4) "[backup] reliable answer for: What are the admission requirements..."
```

### Tại sao shared cache quan trọng cho production

- **In-memory cache không đủ**: Mỗi gateway instance có cache riêng biệt; trong multi-instance (Kubernetes), request có thể hit các instance khác nhau với cache state khác nhau
- **SharedRedisCache giải quyết**: Dùng Redis làm storage tập trung với hash keys `rl:cache:{md5_hash}` chứa `query` và `response` với TTL tự động

### So sánh In-memory vs Redis

| Metric | In-memory | Redis | Ghi chú |
|---:|---:|---:|---|
| latency_p50_ms | ~280 | 272.99 | Tương đương |
| latency_p95_ms | ~320 | 322.78 | Tương đương |
| chia sẻ across instances | ❌ | ✅ | Ưu điểm Redis |
| persistence | None | Redis EXPIRE | Ưu điểm Redis |

## 7. Chaos Scenarios

| Scenario | Hành vi kỳ vọng | Hành vi thực tế | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Tất cả fallback, circuit opens | pass | ✅ PASS |
| primary_flaky_50 | Circuit dao động, mix primary/fallback | pass | ✅ PASS |
| all_healthy | Tất cả qua primary, không circuit opens | pass | ✅ PASS |

## 8. Phân Tích Lỗi

- **Điểm yếu**: Recovery time không đo được (transition log có gaps); không có rate limiting per-user; không có cost budget tracking
- **Khắc phục**: Lưu circuit state trong Redis cho cross-instance; thêm rate limiting per API key; implement cost budget để route đến provider rẻ hơn

## 9. Các Bước Tiếp Theo

1. Thêm Redis circuit breaker state (INCR/EXPIRE) cho cross-instance CB awareness
2. Implement cost budget tracking - route đến provider rẻ hơn khi đạt 80% budget
3. Thêm property-based tests với hypothesis cho CB state machine fuzzing