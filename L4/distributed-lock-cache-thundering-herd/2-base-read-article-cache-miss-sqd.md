# Base sequence — Cache miss đồng loạt, mọi request tự query DB (thundering herd)

Đây là **base**, flow đọc bài viết hiện tại khi cache miss: mỗi request kiểm tra cache, không thấy hoặc thấy hết hạn thì tự đi query DB rồi tự ghi lại cache, không có điều phối giữa các request. Đây chính là kịch bản thundering herd nêu trong đề bài — sau khi publish bài mới hoặc cache hết hạn, hàng trăm request cùng cache miss và cùng dội xuống DB song song.

```mermaid
sequenceDiagram
    actor ReqA as Request A
    actor ReqB as Request B
    actor ReqC as Request C (...hàng trăm request khác)
    participant Cache as Cache Layer
    participant DB as Database

    Note over ReqA,ReqC: Bài viết vừa publish hoặc cache vừa hết hạn, không có CACHE_ENTRY hợp lệ

    par Tất cả request cùng cache miss gần như đồng thời
        ReqA->>Cache: GET article:123
        Cache-->>ReqA: MISS
        ReqB->>Cache: GET article:123
        Cache-->>ReqB: MISS
        ReqC->>Cache: GET article:123
        Cache-->>ReqC: MISS
    end

    par Không ai chờ ai, tất cả tự query DB song song
        ReqA->>DB: SELECT * FROM ARTICLE WHERE id=123
        ReqB->>DB: SELECT * FROM ARTICLE WHERE id=123
        ReqC->>DB: SELECT * FROM ARTICLE WHERE id=123
    end

    DB-->>ReqA: Kết quả
    DB-->>ReqB: Kết quả
    DB-->>ReqC: Kết quả

    par Mỗi request tự ghi lại cache, ghi đè lẫn nhau
        ReqA->>Cache: SET article:123 (ttl_seconds)
        ReqB->>Cache: SET article:123 (ttl_seconds)
        ReqC->>Cache: SET article:123 (ttl_seconds)
    end

    Note over DB: Hàng trăm query trùng lặp dội xuống DB cùng lúc, DB có thể quá tải
```
