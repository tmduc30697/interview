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
