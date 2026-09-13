# Enhance sequence — Fallback khi instance mất kết nối tới Redis coordinator

Đây là **enhance**, flow hoàn toàn mới xử lý network partition giữa instance và Redis. Đáp ứng yêu cầu 5: khi không kết nối được coordinator, instance phải có fallback (tự query DB có giới hạn rate) thay vì treo chờ vô hạn hoặc từ chối toàn bộ request, tránh outage toàn phần.

```mermaid
sequenceDiagram
    actor Req as Request (instance X)
    participant Cache as Cache Layer
    participant Lock as Redis Lock
    participant RateLimiter as Local Rate Limiter (per cache key)
    participant DB as Database

    Req->>Cache: GET article:123
    Cache-->>Req: MISS

    Req->>Lock: SET lock:article:123 NX PX 2000ms
    Lock--xReq: Timeout, không kết nối được Redis (network partition)

    Req->>Req: Phát hiện lỗi kết nối coordinator, chuyển sang chế độ fallback

    Req->>RateLimiter: Kiểm tra rate limit local cho key article:123 (vd tối đa 1 query DB/500ms/instance)

    alt Trong giới hạn rate cho phép
        RateLimiter-->>Req: Cho phép
        Req->>DB: SELECT * FROM ARTICLE WHERE id=123 (fallback, không qua lock)
        DB-->>Req: Kết quả
        Req->>Cache: SET article:123 (best-effort, không đồng bộ với instance khác)
        Req->>Req: INSERT CACHE_MISS_EVENT (action=fallback_no_coordinator)
    else Vượt giới hạn rate (instance khác cũng đang fallback)
        RateLimiter-->>Req: Từ chối query thêm
        alt Còn dữ liệu stale trong cache
            Req-->>Req: Trả dữ liệu stale
        else Không còn gì để trả
            Req-->>Req: Trả lỗi tạm thời "vui lòng thử lại sau vài giây"
        end
    end

    Note over Req,DB: Mỗi instance tự giới hạn rate cục bộ khi mất Redis, tránh toàn bộ fleet cùng dội thẳng xuống DB không kiểm soát
```
