# Leader coordinator cho cache invalidation tránh thundering herd

**Hệ thống:** Hệ thống cache nhiều lớp cho một trang tin tức lớn, khi cache miss đồng loạt (ví dụ sau khi publish bài mới) hàng trăm request cùng dội xuống DB.

**Vai trò của flow:** Dùng leader election/distributed lock để chỉ một request/instance được phép đi warm-up cache/query DB, các request khác chờ hoặc dùng dữ liệu cũ.

**Yêu cầu cụ thể:**
- Chỉ request đầu tiên "giành" được lock cho một cache key mới được query DB và ghi lại cache; các request đến sau trong lúc lock còn giữ phải chờ ngắn (vài trăm ms) hoặc trả stale data nếu có, không tự query DB song song.
- Lock phải tự hết hạn nếu request giữ lock bị treo/crash, để tránh các request khác chờ vô hạn.
- Có cơ chế "single flight" nội bộ trong từng instance kết hợp với lock phân tán giữa các instance để giảm số round-trip tới coordinator.
- Đo lường: số lượng DB query giảm được so với baseline không có coordinator (mục tiêu giảm >90% query trùng lặp trong 1 giây đầu sau cache miss).
- Xử lý network partition giữa instance và Redis: instance phải có fallback (ví dụ tự query DB có giới hạn rate) khi không kết nối được coordinator, tránh outage toàn phần.

---

# Base ERD — Cache đơn giản, cache miss là mỗi request tự query DB

Đây là **base**: trạng thái hệ thống cache *trước khi* có leader coordinator chống thundering herd. Suy luận từ đề bài, base đã có `ARTICLE` là dữ liệu gốc trong DB, và một `CACHE_ENTRY` lưu bản render sẵn của article theo cache key (thường là `article:{id}`) với TTL cố định. Khi cache miss (hết hạn hoặc chưa có), base chưa có bất kỳ cơ chế điều phối nào — request nào cũng tự query DB rồi tự ghi lại cache, không ai chờ ai.

```mermaid
erDiagram
    ARTICLE ||--o| CACHE_ENTRY : "được cache dưới dạng"

    ARTICLE {
        string id PK
        string title
        text content
        datetime published_at
        datetime updated_at
    }
    CACHE_ENTRY {
        string cache_key PK
        string article_id FK
        text rendered_value
        int ttl_seconds
        datetime cached_at
    }
```

---

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

---

# Enhance ERD — Thêm ghi nhận sự kiện cache miss để đo lường hiệu quả coordinator

Đây là ERD **sau khi** enhance được áp dụng lên base. `ARTICLE` và `CACHE_ENTRY` giữ nguyên như base (bản thân distributed lock là 1 key ephemeral trên Redis dạng `lock:article:{id}` có TTL, không phải dữ liệu bền vững nên không xuất hiện như entity riêng). Điểm mới là entity `CACHE_MISS_EVENT`, ứng trực tiếp với các yêu cầu:

- `CACHE_MISS_EVENT.role` (leader/follower) và `action` (`queried_db`/`waited_lock`/`served_stale`) — phản ánh kết quả của cơ chế lock ở yêu cầu 1 (chỉ leader query DB, follower chờ hoặc dùng stale data).
- `CACHE_MISS_EVENT.action=fallback_no_coordinator` — phản ánh yêu cầu 5 (fallback khi mất kết nối tới Redis coordinator).
- Toàn bộ bảng `CACHE_MISS_EVENT` chính là nguồn dữ liệu để tính chỉ số ở yêu cầu 4 (đo % giảm số DB query trùng lặp so với baseline).
- `CACHE_ENTRY.stale_serve_deadline` — hỗ trợ việc follower vẫn trả được dữ liệu cũ (stale) trong lúc chờ leader warm-up xong, thay vì phải chờ trắng.

```mermaid
erDiagram
    ARTICLE ||--o| CACHE_ENTRY : "được cache dưới dạng"
    CACHE_ENTRY ||--o{ CACHE_MISS_EVENT : "ghi nhận mỗi lần miss"

    ARTICLE {
        string id PK
        string title
        text content
        datetime published_at
        datetime updated_at
    }
    CACHE_ENTRY {
        string cache_key PK
        string article_id FK
        text rendered_value
        int ttl_seconds
        datetime cached_at
        datetime stale_serve_deadline "vẫn được phép trả stale tới thời điểm này"
    }
    CACHE_MISS_EVENT {
        string id PK
        string cache_key FK
        string instance_id
        string role "leader|follower"
        string action "queried_db|waited_lock|served_stale|fallback_no_coordinator"
        int wait_time_ms
        datetime occurred_at
    }
```

---

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

---

# Enhance sequence — Singleflight nội bộ instance kết hợp distributed lock

Đây là **enhance**, flow hoàn toàn mới mô tả tầng singleflight bên trong 1 instance, đứng trước distributed lock ở tầng giữa các instance. Đáp ứng yêu cầu 3: nhiều request đồng thời trong cùng 1 instance cho cùng cache key chỉ tạo ra đúng 1 lời gọi tới coordinator (Redis), giảm số round-trip không cần thiết tới Redis khi vốn dĩ chúng đã có thể gộp chung kết quả ngay trong process.

```mermaid
sequenceDiagram
    actor Req1 as Request 1 (instance X)
    actor Req2 as Request 2 (instance X, cùng key)
    actor Req3 as Request 3 (instance X, cùng key)
    participant SF as Singleflight (in-process, instance X)
    participant Lock as Redis Lock
    participant DB as Database
    participant Cache as Cache Layer

    par 3 request cùng cache miss trong cùng 1 instance
        Req1->>SF: Cần dữ liệu article:123
        Req2->>SF: Cần dữ liệu article:123
        Req3->>SF: Cần dữ liệu article:123
    end

    SF->>SF: Phát hiện cùng key đang inflight, chỉ tạo 1 goroutine/task thực thi chung
    Note over SF: Req2, Req3 được gắn vào cùng 1 future/promise với Req1, không tạo request riêng

    SF->>Lock: (chỉ 1 lần) SET lock:article:123 NX PX 2000ms
    Lock-->>SF: Giành lock thành công

    SF->>DB: (chỉ 1 lần) SELECT * FROM ARTICLE WHERE id=123
    DB-->>SF: Kết quả

    SF->>Cache: SET article:123
    SF->>Lock: DEL lock:article:123

    SF-->>Req1: Trả kết quả
    SF-->>Req2: Trả kết quả (từ cùng 1 future, không gọi Redis/DB riêng)
    SF-->>Req3: Trả kết quả (từ cùng 1 future, không gọi Redis/DB riêng)

    Note over SF,Lock: Thay vì 3 lần round-trip tới Redis, singleflight gộp lại chỉ còn đúng 1 lần
```

---

# Enhance sequence — Đo lường mức giảm DB query trùng lặp so với baseline

Đây là **enhance**, flow hoàn toàn mới mô tả cách hệ thống đo lường hiệu quả của coordinator sau khi triển khai. Đáp ứng yêu cầu 4: cần con số cụ thể chứng minh giảm được >90% query trùng lặp trong 1 giây đầu sau cache miss, dựa trên dữ liệu `CACHE_MISS_EVENT` được ghi lại ở flow acquire-lock.

```mermaid
sequenceDiagram
    participant App as Application Instances
    participant DB2 as CACHE_MISS_EVENT store
    participant Job as Metrics Aggregator Job
    actor SRE as SRE/Dev xem dashboard

    Note over App,DB2: Trong flow read-article-cache-miss, mỗi request đều ghi 1 CACHE_MISS_EVENT (role, action, wait_time_ms)

    App->>DB2: INSERT CACHE_MISS_EVENT (role=leader, action=queried_db)
    App->>DB2: INSERT CACHE_MISS_EVENT (role=follower, action=waited_lock)
    App->>DB2: INSERT CACHE_MISS_EVENT (role=follower, action=served_stale)

    loop Mỗi phút
        Job->>DB2: SELECT cache_key, COUNT(*) FILTER (action=queried_db) AS actual_db_queries, COUNT(*) AS total_requests GROUP BY cache_key trong cửa sổ 1s sau miss đầu tiên
        DB2-->>Job: Kết quả tổng hợp theo từng đợt cache miss

        Job->>Job: Tính reduction_rate = 1 - (actual_db_queries / total_requests_ước_tính_nếu_không_có_coordinator)
    end

    Job-->>SRE: Publish dashboard, vd "Đợt cache miss bài viết 123: 420 request, chỉ 1 query DB thật, giảm 99.7% so với baseline"

    Note over Job,SRE: Baseline tham chiếu là số liệu đo được ở flow base (mỗi request 1 query), so sánh trực tiếp để xác nhận đạt mục tiêu giảm >90%
```

---

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
