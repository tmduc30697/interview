# Enhance ERD — Cache offline-first bằng IndexedDB

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (chỉ có PROJECT/RECORD ở server), phần enhance thêm 6 nhóm entity mới ở phía client (IndexedDB) và đo lường, ứng trực tiếp với các yêu cầu trong đề bài:

- `CACHED_RECORD` (mới, IndexedDB) — bản snapshot record để hiển thị ngay, kèm `last_synced_at` và `is_stale` để biết đang hiển thị dữ liệu cũ hay đã đồng bộ — yêu cầu 1.
- `SCHEMA_VERSION` (mới, meta trong IndexedDB) — theo dõi version schema hiện tại của database cục bộ, phục vụ `onupgradeneeded` migration — yêu cầu 2.
- `OUTBOX_ENTRY` (mới, IndexedDB) — hàng đợi thao tác sửa/xoá thực hiện lúc offline, có `order_seq` để replay đúng thứ tự — yêu cầu 3.
- `SYNC_CONFLICT` (mới) — ghi nhận khi 1 outbox entry replay lên server nhưng bản ghi đã bị người khác sửa trong lúc offline — yêu cầu 3.
- `STORAGE_QUOTA_STATE` + `EVICTION_LOG` (mới) — theo dõi dung lượng IndexedDB đã dùng so với quota per-origin, và lịch sử eviction theo project ít dùng nhất (LRU) — yêu cầu 4.
- `CACHE_METRIC` (mới) — tỉ lệ cache hit so với phải gọi API, và độ trễ từ lúc hiển thị cache tới lúc đồng bộ xong dữ liệu mới nhất — yêu cầu 5.

```mermaid
erDiagram
    PROJECT ||--o{ RECORD : contains
    USER ||--o{ RECORD : edits
    RECORD ||--o| CACHED_RECORD : "cached as (IndexedDB)"
    PROJECT ||--o{ CACHED_RECORD : "grouped for eviction"
    USER ||--o{ OUTBOX_ENTRY : "queues while offline"
    RECORD ||--o{ OUTBOX_ENTRY : "target of"
    OUTBOX_ENTRY ||--o| SYNC_CONFLICT : "may raise"
    PROJECT ||--o{ EVICTION_LOG : "evicted when quota near full"
    PROJECT ||--o{ CACHE_METRIC : "measured per"

    USER {
        string id PK
        string display_name
    }
    PROJECT {
        string id PK
        string name
    }
    RECORD {
        string id PK
        string project_id FK
        string type "task | ticket | contact"
        string data
        string updated_by FK
        datetime updated_at
    }
    CACHED_RECORD {
        string id PK
        string record_id FK
        string project_id FK
        string data_snapshot
        datetime cached_at
        datetime last_synced_at
        boolean is_stale
        datetime last_accessed_at
    }
    SCHEMA_VERSION {
        int db_version PK
        string migration_notes
        datetime migrated_at
    }
    OUTBOX_ENTRY {
        string id PK
        string record_id FK
        string user_id FK
        string action "update | delete"
        string payload
        int order_seq
        string status "pending | synced | conflict"
        datetime created_at
    }
    SYNC_CONFLICT {
        string id PK
        string outbox_entry_id FK
        string server_version_data
        string local_version_data
        string resolution "keep_server | keep_local | manual_merge"
        datetime detected_at
    }
    STORAGE_QUOTA_STATE {
        string origin PK
        int used_bytes
        int quota_bytes
        string eviction_policy "LRU_by_project"
        datetime checked_at
    }
    EVICTION_LOG {
        string id PK
        string project_id FK
        int freed_bytes
        string reason "quota_near_full"
        datetime evicted_at
    }
    CACHE_METRIC {
        string id PK
        string project_id FK
        date window_date
        int cache_hit_count
        int cache_miss_count
        int avg_stale_to_synced_ms
    }
```
