# ERD — Enhance (sau khi có pipeline transcode đa độ phân giải)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 6 yêu cầu của đề bài — resumable upload theo chunk, transcode song song nhiều độ phân giải, sinh thumbnail + manifest HLS/DASH, retry đúng phần thiếu, progressive availability, và cleanup có khả năng re-transcode. So với base, `VIDEO` không còn gắn 1 file kết quả duy nhất mà tách thành nhiều `RENDITION` độc lập, publish được ngay khi có rendition đầu tiên thay vì chờ toàn bộ.

```mermaid
erDiagram
    USER ||--o{ VIDEO : uploads
    VIDEO ||--o{ UPLOAD_CHUNK : "uploaded via"
    VIDEO ||--o{ RENDITION : "transcoded into"
    VIDEO ||--o{ THUMBNAIL : "generated as"
    VIDEO ||--o| MANIFEST : "packaged as"
    VIDEO ||--o| RETENTION_POLICY : "governed by"

    USER {
        string user_id PK
        string display_name
        string status
    }

    VIDEO {
        string video_id PK
        string user_id FK
        string raw_storage_path
        string status
        boolean raw_deleted
        datetime uploaded_at
        datetime published_at
    }

    UPLOAD_CHUNK {
        string chunk_id PK
        string video_id FK
        int chunk_index
        string status
        int size_bytes
    }

    RENDITION {
        string rendition_id PK
        string video_id FK
        string resolution
        string status
        string storage_path
        string failed_segment_ref
    }

    THUMBNAIL {
        string thumbnail_id PK
        string video_id FK
        string storage_path
        int frame_position
        boolean is_selected
    }

    MANIFEST {
        string manifest_id PK
        string video_id FK
        string format
        string storage_path
    }

    RETENTION_POLICY {
        string policy_id PK
        string video_id FK
        int retain_days
        boolean original_cleaned_up
        boolean retranscode_possible
    }
```
