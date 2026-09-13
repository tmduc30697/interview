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
