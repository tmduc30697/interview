# Enhance sequence — Sale boundary prewarm

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 3 của đề bài: pre-warm cache trước thời điểm giờ flash sale bắt đầu/kết thúc vài giây, tránh hàng loạt cache cùng miss đồng thời đúng lúc chốt giờ (cache stampede).

```mermaid
sequenceDiagram
    participant Scheduler as Sale Schedule Scheduler
    participant Job as SALE_WARMUP_JOB store
    participant DB as PRODUCT store
    participant PriceCache as PRICE_CACHE_ENTRY store

    Scheduler->>Scheduler: Theo dõi sale_start_at/sale_end_at của các sản phẩm sắp tới
    Scheduler->>Job: Tạo SALE_WARMUP_JOB (scheduled_change_at, warmup_started_at = scheduled_change_at - vài giây)

    Note over Job: Đợi tới warmup_started_at (vài giây trước giờ chốt)

    Job->>DB: Tính trước giá hiệu lực sẽ áp dụng ngay sau thời điểm chốt (sale_price hoặc giá gốc)
    Job->>PriceCache: Ghi sẵn PRICE_CACHE_ENTRY mới cho giá sắp có hiệu lực, cached_at đặt đúng thời điểm chuyển
    PriceCache-->>Job: Cache đã sẵn sàng trước giờ chốt

    Note over Job,PriceCache: Khi đồng hồ chạm sale_start_at/sale_end_at, cache đã có sẵn giá trị mới — request đọc ngay sau đó đều hit cache, không có làn sóng miss đồng thời dội xuống DB

    Job->>Job: status=completed
```
