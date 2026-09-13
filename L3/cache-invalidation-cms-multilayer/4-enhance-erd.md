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
