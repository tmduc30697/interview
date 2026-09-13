# Cache nhiều tầng cho CMS/blog

**Hệ thống:** Hệ thống CMS/blog phục vụ bài viết qua nhiều tầng cache chồng lên nhau: CDN edge cache (gần user), cache tầng ứng dụng (application/object cache), và cache kết quả query DB (query cache) — mỗi tầng có TTL và cơ chế invalidate riêng.

**Vai trò của flow:** Đảm bảo khi 1 bài viết được sửa/xóa/publish, việc invalidate phải lan đúng và đủ qua toàn bộ các tầng cache, không chỉ dừng ở tầng gần nhất (application cache) mà bỏ sót CDN edge hoặc query cache khiến người đọc vẫn thấy nội dung cũ dù tầng app đã đúng.

**Yêu cầu cụ thể:**
- Khi biên tập viên sửa nội dung bài viết và publish lại, invalidation phải được gửi tới cả 3 tầng theo đúng thứ tự phụ thuộc (query cache trước, application cache sau, rồi purge CDN edge) — nếu purge CDN trước mà tầng dưới chưa cập nhật, request tiếp theo tới edge sẽ pull lại đúng nội dung cũ từ application cache và cache lại nó ở edge, coi như invalidate CDN vô nghĩa.
- Purge CDN edge cache thường không đồng bộ ngay lập tức trên toàn bộ điểm PoP (point of presence) toàn cầu — phải chấp nhận và công bố rõ "khoảng lan truyền" (propagation window) thực tế, đồng thời có cơ chế xác nhận purge đã tới đủ số PoP quan trọng trước khi coi invalidation hoàn tất.
- Với bài viết bị xóa hẳn (không chỉ sửa), phải đảm bảo tầng query cache (thường cache theo query như "danh sách bài mới nhất") cũng được invalidate, không chỉ cache theo ID bài viết đơn lẻ — nếu bỏ sót, bài đã xóa vẫn xuất hiện trong danh sách trang chủ dù trang chi tiết đã trả 404 đúng.
- Xử lý trường hợp một tầng cache invalidate thất bại (ví dụ API purge CDN timeout hoặc trả lỗi) — phải có retry với theo dõi trạng thái, và log rõ tầng nào invalidate thành công/thất bại để vận hành biết còn tầng nào đang phục vụ nội dung cũ, tránh tình trạng "tưởng đã invalidate hết" nhưng thực ra một tầng vẫn stale.
- Đo lường: với mỗi lần publish/sửa bài, track được "end-to-end staleness" — thời gian từ lúc publish tới lúc cả 3 tầng đều phản ánh đúng nội dung mới, phân theo từng tầng để biết tầng nào đang là nút thắt cổ chai của độ trễ invalidate.

---

# Base ERD — CMS đa tầng cache trước khi có orchestration invalidation

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có cơ chế điều phối invalidation xuyên 3 tầng. Đề bài mô tả hệ thống đã có sẵn 3 tầng cache (CDN edge, application cache, query cache) — nên base cần đủ: bài viết, cache tầng ứng dụng theo article_id, cache tầng query (theo danh sách như "latest_posts"), và 1 request purge CDN thô sơ (fire-and-forget, không xác nhận PoP). Chưa có entity nào phục vụ điều phối thứ tự/retry/đo staleness — những thứ đó là phần enhance.

```mermaid
erDiagram
    ARTICLE ||--o| APP_CACHE_ENTRY : "cached as"
    ARTICLE ||--o{ CDN_PURGE_REQUEST : triggers

    ARTICLE {
        string id PK
        string title
        string content
        string status "draft | published | deleted"
        datetime updated_at
    }
    APP_CACHE_ENTRY {
        string cache_key PK "article_id"
        string value
        datetime cached_at
        int ttl_seconds
    }
    QUERY_CACHE_ENTRY {
        string cache_key PK "vd latest_posts, category:tech"
        string value
        datetime cached_at
        int ttl_seconds
    }
    CDN_PURGE_REQUEST {
        string id PK
        string article_id FK
        datetime requested_at
        string status "sent"
    }
```

---

# Base sequence — Publish article (invalidation chưa đúng thứ tự)

Đây là **base**, flow "Biên tập viên sửa và publish lại bài viết" ở trạng thái hiện tại — chỉ invalidate application cache đúng, còn CDN thì bắn đi không chờ xác nhận và không theo thứ tự phụ thuộc. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 1 và 2 của đề bài chính là sửa lại đúng vấn đề thứ tự và xác nhận purge ở đây.

```mermaid
sequenceDiagram
    actor Editor
    participant CMS as CMS App
    participant DB as ARTICLE store
    participant AppCache as APP_CACHE_ENTRY store
    participant CDN as CDN Edge (nhiều PoP)

    Editor->>CMS: Sửa nội dung, bấm Publish
    CMS->>DB: Cập nhật ARTICLE (content, updated_at)
    par Không có thứ tự phụ thuộc rõ ràng
        CMS->>AppCache: Invalidate APP_CACHE_ENTRY(article_id)
    and
        CMS->>CDN: Gọi API purge CDN cho article_id (không chờ app cache invalidate xong)
        CDN-->>CMS: Trả 200 ngay (API nhận request, chưa chắc đã lan hết PoP)
    end
    CMS-->>Editor: "Publish thành công" (coi purge CDN là xong ngay khi API trả 200)
    Note over CMS,CDN: Không invalidate QUERY_CACHE_ENTRY (vd latest_posts) — vẫn giữ bản cũ. Vì 2 nhánh chạy song song không có thứ tự, có thể xảy ra race: 1 PoP CDN miss ngay lúc app cache vẫn còn dữ liệu cũ, PoP đó pull lại và cache lại bản cũ — coi như purge vô nghĩa
```

---

# Base sequence — Delete article (bỏ sót query cache)

Đây là **base**, flow "Xóa hẳn bài viết" ở trạng thái hiện tại — chỉ invalidate cache theo article_id, không đụng tới các query cache theo danh sách. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 3 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    actor Editor
    participant CMS as CMS App
    participant DB as ARTICLE store
    participant AppCache as APP_CACHE_ENTRY store
    participant QueryCache as QUERY_CACHE_ENTRY store
    participant CDN as CDN Edge

    Editor->>CMS: Xóa hẳn bài viết
    CMS->>DB: Cập nhật ARTICLE.status=deleted
    CMS->>AppCache: Invalidate APP_CACHE_ENTRY(article_id)
    CMS->>CDN: Purge CDN cho trang chi tiết bài viết (article_id)
    CMS-->>Editor: "Xóa thành công"
    Note over QueryCache: Không được đụng tới — QUERY_CACHE_ENTRY("latest_posts") vẫn còn chứa bài đã xóa
    Note over CMS,QueryCache: Kết quả: trang chi tiết trả 404 đúng, nhưng bài đã xóa vẫn xuất hiện trong danh sách trang chủ tới khi TTL của query cache tự hết hạn
```

---

# Enhance ERD — sau khi có orchestration invalidation đa tầng

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `INVALIDATION_JOB` + `INVALIDATION_STEP` (mới) — điều phối invalidation theo đúng thứ tự (query cache → app cache → CDN), mỗi bước theo dõi trạng thái/retry riêng thay vì bắn song song như base.
- `CDN_PURGE_REQUEST` thêm `target_pop_count`/`confirmed_pop_count`/`propagation_window_ends_at` — chỉ coi purge hoàn tất khi đủ số PoP quan trọng xác nhận, không còn coi API trả 200 là xong ngay.
- `QUERY_CACHE_INVALIDATION_RULE` (mới) — khai báo rõ query cache nào (vd "latest_posts") phải bị invalidate khi có publish/delete, để không bỏ sót như base.
- `STALENESS_METRIC` (mới) — đo end-to-end staleness từ lúc publish tới lúc từng tầng phản ánh đúng nội dung mới.

```mermaid
erDiagram
    ARTICLE ||--o| APP_CACHE_ENTRY : "cached as"
    ARTICLE ||--o{ INVALIDATION_JOB : triggers
    INVALIDATION_JOB ||--o{ INVALIDATION_STEP : "consists of"
    INVALIDATION_JOB ||--o{ STALENESS_METRIC : measures
    INVALIDATION_STEP ||--o| CDN_PURGE_REQUEST : "uses (khi layer=cdn_edge)"
    QUERY_CACHE_INVALIDATION_RULE ||--o{ INVALIDATION_STEP : "áp dụng cho (khi layer=query_cache)"

    ARTICLE {
        string id PK
        string title
        string content
        string status "draft | published | deleted"
        datetime updated_at
    }
    APP_CACHE_ENTRY {
        string cache_key PK "article_id"
        string value
        datetime cached_at
        int ttl_seconds
    }
    QUERY_CACHE_ENTRY {
        string cache_key PK "vd latest_posts, category:tech"
        string value
        datetime cached_at
        int ttl_seconds
    }
    QUERY_CACHE_INVALIDATION_RULE {
        string id PK
        string query_cache_key
        string invalidate_on "publish | delete | update"
        string description
    }
    INVALIDATION_JOB {
        string id PK
        string article_id FK
        string trigger_type "publish | delete"
        string status "in_progress | completed | partially_stale"
        datetime created_at
        datetime completed_at
    }
    INVALIDATION_STEP {
        string id PK
        string invalidation_job_id FK
        string layer "query_cache | app_cache | cdn_edge"
        string status "pending | success | failed | retrying"
        int attempt_count
        datetime started_at
        datetime completed_at
    }
    CDN_PURGE_REQUEST {
        string id PK
        string invalidation_step_id FK
        string article_id FK
        int target_pop_count
        int confirmed_pop_count
        datetime requested_at
        datetime propagation_window_ends_at
        string status "pending | confirmed | failed"
    }
    STALENESS_METRIC {
        string id PK
        string invalidation_job_id FK
        string layer
        datetime published_at
        datetime layer_consistent_at
        int staleness_ms
    }
```

---

# Enhance sequence — Publish article (đúng thứ tự, có xác nhận CDN, đo staleness)

Đây là **enhance**, cùng flow "Publish article" đã có ở base nhưng nay thay đổi theo yêu cầu 1, 2 và 5 của đề bài: 3 tầng được invalidate theo đúng thứ tự phụ thuộc (query cache → app cache → CDN), CDN chỉ coi là xong khi đủ số PoP xác nhận trong propagation window, và mỗi tầng được đo staleness riêng.

```mermaid
sequenceDiagram
    actor Editor
    participant CMS as CMS App
    participant DB as ARTICLE store
    participant Job as INVALIDATION_JOB store
    participant QueryCache as QUERY_CACHE_ENTRY store
    participant AppCache as APP_CACHE_ENTRY store
    participant CDN as CDN Edge (nhiều PoP)

    Editor->>CMS: Sửa nội dung, bấm Publish
    CMS->>DB: Cập nhật ARTICLE (content, updated_at)
    CMS->>Job: Tạo INVALIDATION_JOB (trigger_type=publish)

    Job->>QueryCache: Bước 1 — invalidate theo QUERY_CACHE_INVALIDATION_RULE (vd latest_posts)
    QueryCache-->>Job: INVALIDATION_STEP(layer=query_cache) = success
    Job->>Job: Ghi STALENESS_METRIC(layer=query_cache, layer_consistent_at=now)

    Job->>AppCache: Bước 2 (chỉ chạy sau khi bước 1 xong) — invalidate APP_CACHE_ENTRY(article_id)
    AppCache-->>Job: INVALIDATION_STEP(layer=app_cache) = success
    Job->>Job: Ghi STALENESS_METRIC(layer=app_cache, layer_consistent_at=now)

    Job->>CDN: Bước 3 (chỉ chạy sau khi bước 2 xong) — purge CDN cho article_id
    CDN-->>Job: Tạo CDN_PURGE_REQUEST (target_pop_count, propagation_window_ends_at)
    loop Trong propagation window
        Job->>CDN: Kiểm tra confirmed_pop_count
        CDN-->>Job: Cập nhật confirmed_pop_count
    end
    alt confirmed_pop_count đạt ngưỡng trước khi hết window
        Job->>Job: INVALIDATION_STEP(layer=cdn_edge) = success
        Job->>Job: Ghi STALENESS_METRIC(layer=cdn_edge, layer_consistent_at=now)
        Job->>Job: INVALIDATION_JOB status=completed
        Job-->>Editor: "Publish thành công, đã invalidate đủ 3 tầng"
    else Hết propagation window mà chưa đủ PoP xác nhận
        Job->>Job: INVALIDATION_STEP(layer=cdn_edge) = failed → chuyển sang flow retry
        Job->>Job: INVALIDATION_JOB status=partially_stale
        Job-->>Editor: "Publish thành công ở 2 tầng, CDN đang xử lý tiếp"
    end
```

---

# Enhance sequence — Delete article (invalidate cả query cache dạng danh sách)

Đây là **enhance**, cùng flow "Delete article" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 3 của đề bài: dùng `QUERY_CACHE_INVALIDATION_RULE` để xác định và invalidate đúng các query cache dạng danh sách (không chỉ cache theo article_id), đi theo cùng thứ tự query cache → app cache → CDN như flow publish.

```mermaid
sequenceDiagram
    actor Editor
    participant CMS as CMS App
    participant DB as ARTICLE store
    participant Job as INVALIDATION_JOB store
    participant Rule as QUERY_CACHE_INVALIDATION_RULE store
    participant QueryCache as QUERY_CACHE_ENTRY store
    participant AppCache as APP_CACHE_ENTRY store
    participant CDN as CDN Edge

    Editor->>CMS: Xóa hẳn bài viết
    CMS->>DB: Cập nhật ARTICLE.status=deleted
    CMS->>Job: Tạo INVALIDATION_JOB (trigger_type=delete)

    Job->>Rule: Tra toàn bộ rule có invalidate_on=delete (vd latest_posts, category listing chứa bài này)
    Rule-->>Job: Danh sách query_cache_key cần invalidate
    Job->>QueryCache: Invalidate từng QUERY_CACHE_ENTRY theo danh sách trên (không chỉ cache theo article_id)
    QueryCache-->>Job: INVALIDATION_STEP(layer=query_cache) = success

    Job->>AppCache: Invalidate APP_CACHE_ENTRY(article_id)
    AppCache-->>Job: INVALIDATION_STEP(layer=app_cache) = success

    Job->>CDN: Purge CDN cho trang chi tiết bài viết
    CDN-->>Job: Xác nhận đủ PoP trong propagation window (như flow Publish)
    Job->>Job: INVALIDATION_JOB status=completed

    Job-->>Editor: "Xóa thành công — bài viết đã biến mất khỏi cả trang chi tiết lẫn danh sách trang chủ"
```

---

# Enhance sequence — Invalidation failure & retry

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (ở base, purge CDN fire-and-forget, không ai biết nó có thất bại hay không). Đáp ứng yêu cầu thứ 4 của đề bài: khi 1 tầng invalidate thất bại (vd API purge CDN timeout), phải retry có theo dõi trạng thái và log rõ tầng nào thành công/thất bại.

```mermaid
sequenceDiagram
    participant Job as INVALIDATION_JOB store
    participant Step as INVALIDATION_STEP store
    participant CDN as CDN Edge API
    participant Ops as Ops Dashboard / Alerting

    Note over Step: INVALIDATION_STEP(layer=cdn_edge) vừa fail (API timeout hoặc trả lỗi)
    Step->>Step: status=retrying, attempt_count += 1
    Job->>Ops: Log ngay: "layer=cdn_edge failed, attempt=1, các layer khác vẫn giữ trạng thái success"

    loop Retry với backoff, tối đa N lần
        Step->>CDN: Gọi lại API purge
        alt Thành công
            CDN-->>Step: Xác nhận đủ PoP
            Step->>Step: status=success, completed_at=now
            Job->>Job: Kiểm tra toàn bộ INVALIDATION_STEP → nếu đủ 3 tầng success, INVALIDATION_JOB status=completed
            Job->>Ops: Log "layer=cdn_edge recovered sau retry"
        else Vẫn thất bại
            CDN-->>Step: Timeout/lỗi tiếp
            Step->>Step: attempt_count += 1
        end
    end

    alt Hết số lần retry cho phép mà vẫn fail
        Step->>Step: status=failed (dừng retry tự động)
        Job->>Job: INVALIDATION_JOB status=partially_stale
        Job->>Ops: Cảnh báo rõ ràng: "article X — query_cache: success, app_cache: success, cdn_edge: failed — vẫn còn nội dung cũ ở CDN"
        Ops->>Ops: Vận hành biết chính xác tầng nào đang stale để xử lý thủ công thay vì tưởng đã invalidate hết
    end
```
