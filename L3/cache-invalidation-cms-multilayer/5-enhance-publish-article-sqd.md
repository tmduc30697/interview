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
