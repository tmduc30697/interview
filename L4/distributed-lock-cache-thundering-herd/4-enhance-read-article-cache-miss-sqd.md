# Enhance sequence — Chỉ leader query DB, follower chờ ngắn hoặc trả stale data

Đây là **enhance** của flow `read-article-cache-miss` đã có ở base. So với base (mọi request tự query DB song song), enhance thêm distributed lock trên Redis theo từng cache key: chỉ request giành được lock (leader) mới query DB và warm-up cache, các request khác (follower) chờ ngắn hoặc trả ngay dữ liệu cũ nếu có. Đáp ứng yêu cầu 1 (chỉ leader query DB, follower chờ vài trăm ms hoặc dùng stale data, không tự query song song) và yêu cầu 2 (lock có TTL, tự hết hạn nếu leader bị treo/crash).

```mermaid
sequenceDiagram
    actor ReqA as Request A (thành leader)
    actor ReqB as Request B (follower)
    participant Cache as Cache Layer
    participant Lock as Redis Lock (lock:article:123, TTL 2s)
    participant DB as Database

    par Cache miss gần như đồng thời
        ReqA->>Cache: GET article:123
        Cache-->>ReqA: MISS (hoặc còn stale_serve_deadline)
        ReqB->>Cache: GET article:123
        Cache-->>ReqB: MISS (hoặc còn stale_serve_deadline)
    end

    ReqA->>Lock: SET lock:article:123 NX PX 2000ms
    Lock-->>ReqA: Giành lock thành công, trở thành leader

    ReqB->>Lock: SET lock:article:123 NX PX 2000ms
    Lock-->>ReqB: Thất bại, lock đang bị leader giữ

    alt Còn dữ liệu cache cũ trong stale_serve_deadline
        ReqB-->>ReqB: Trả ngay dữ liệu stale, không chờ
    else Không có dữ liệu cũ để trả
        ReqB->>ReqB: Chờ ngắn (vd 300ms) rồi kiểm tra lại cache
    end

    ReqA->>DB: SELECT * FROM ARTICLE WHERE id=123
    DB-->>ReqA: Kết quả

    ReqA->>Cache: SET article:123 (ttl_seconds, stale_serve_deadline mới)
    ReqA->>Lock: DEL lock:article:123 (release ngay sau khi warm-up xong)

    ReqB->>Cache: GET article:123 (sau khi chờ 300ms)
    Cache-->>ReqB: HIT, dữ liệu mới do leader vừa ghi

    Note over Lock: Nếu leader bị treo/crash không kịp DEL, lock tự hết hạn sau 2000ms, request khác giành lock và trở thành leader mới
```
