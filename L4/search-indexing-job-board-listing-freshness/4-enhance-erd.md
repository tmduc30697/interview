# ERD — Enhance (sau khi cải thiện độ tươi mới)

Đây là **enhance**: ERD base cộng với `SKILL_SYNONYM_MAP` để ánh xạ các cách gõ kỹ năng khác nhau về cùng khái niệm, và `JOB_INDEX_DOCUMENT` được bổ sung cơ chế đồng bộ sự kiện tức thời thay vì batch định kỳ. So với base, việc đồng bộ index không còn qua `Periodic Sync Job` mà qua `INDEX_SYNC_EVENT` được xử lý gần như ngay lập tức, và `JOB_INDEX_DOCUMENT` có thêm `skills_normalized` để phục vụ autocomplete theo tên kỹ năng chuẩn.

```mermaid
erDiagram
    EMPLOYER ||--o{ JOB_POSTING : posts
    JOB_POSTING ||--o{ INDEX_SYNC_EVENT : "triggers on change"
    INDEX_SYNC_EVENT ||--o| JOB_INDEX_DOCUMENT : updates
    SKILL_SYNONYM_MAP ||--o{ JOB_INDEX_DOCUMENT : "normalizes skills for"

    EMPLOYER {
        string employer_id PK
        string company_name
    }

    JOB_POSTING {
        string job_id PK
        string employer_id FK
        string title
        string skills_text
        string location
        decimal salary_min
        decimal salary_max
        boolean salary_visible
        string status
        date expires_at
        string timezone
        datetime updated_at
    }

    JOB_INDEX_DOCUMENT {
        string job_id PK
        string title
        string skills_text
        string skills_normalized
        string location
        decimal salary_min
        decimal salary_max
        boolean salary_visible
        string status
        datetime expires_at
        datetime last_synced_at
    }

    SKILL_SYNONYM_MAP {
        string mapping_id PK
        string raw_term
        string canonical_skill_id
    }

    INDEX_SYNC_EVENT {
        string event_id PK
        string job_id FK
        string change_type
        string status
        datetime created_at
        datetime processed_at
    }
```
