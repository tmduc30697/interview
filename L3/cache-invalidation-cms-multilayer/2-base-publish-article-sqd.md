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
