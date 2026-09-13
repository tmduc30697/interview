# Enhance sequence — Đo cache hit rate và độ trễ đồng bộ

Đây là **enhance**, flow hoàn toàn mới, đáp ứng yêu cầu 5 của đề bài: đo tỉ lệ cache hit của IndexedDB so với phải gọi API, và độ trễ giữa lúc hiển thị từ cache tới lúc đồng bộ xong dữ liệu mới nhất, để biết cache có thực sự giúp ích hay đang gây hiển thị sai kéo dài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (CACHED_RECORD)
    participant API as List API
    participant Metric as CACHE_METRIC
    actor Analyst as Kỹ sư / Product Analyst

    User->>App: Mở trang danh sách
    App->>IDB: Kiểm tra có CACHED_RECORD cho project này không

    alt Có cache, dùng được ngay (cache hit)
        IDB-->>App: Trả về dữ liệu cache, ghi lại thời điểm hiển thị cache_shown_at
        App->>Metric: Tăng cache_hit_count cho project
    else Không có cache, phải chờ API (cache miss)
        App->>Metric: Tăng cache_miss_count cho project
    end

    App->>API: Gọi API lấy dữ liệu mới nhất (ngầm nếu cache hit, đồng bộ nếu cache miss)
    API-->>App: Trả về dữ liệu mới nhất tại thời điểm synced_at

    alt Trước đó là cache hit
        App->>Metric: Tính stale_to_synced_ms = synced_at trừ cache_shown_at, cộng dồn vào avg_stale_to_synced_ms
    end

    loop Định kỳ
        Metric-->>Analyst: Báo cáo cache_hit_count, cache_miss_count, avg_stale_to_synced_ms theo từng project
        Analyst->>Analyst: Đánh giá, nếu avg_stale_to_synced_ms quá lớn thì cache đang hiển thị sai kéo dài, cần xem lại tần suất revalidate
    end
```
