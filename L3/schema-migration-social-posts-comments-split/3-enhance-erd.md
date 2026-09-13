# ERD — Enhance (sau khi tách bảng comments)

Đây là **enhance**: ERD base cộng với bảng `COMMENT` quan hệ, tự tham chiếu để hỗ trợ trả lời lồng nhiều cấp, checkpoint backfill theo bài viết, và báo cáo đối soát thứ tự/nội dung. So với base, `POST` giữ nguyên `comments_json` (chưa drop, vẫn dual-write) trong khi `COMMENT` là bảng hoàn toàn mới cho phép mỗi comment là 1 dòng riêng, có `parent_comment_id` tự tham chiếu để biểu diễn reply lồng cấp — điều JSON cũ không hỗ trợ tốt.

```mermaid
erDiagram
    USER ||--o{ POST : creates
    USER ||--o{ COMMENT : writes
    POST ||--o{ COMMENT : "has (new, dual-written)"
    COMMENT ||--o{ COMMENT : "replies to (self-referencing)"
    POST ||--o| ORDER_CONSISTENCY_CHECK : "validated by"

    USER {
        string user_id PK
        string username
        string email
    }

    POST {
        string post_id PK
        string user_id FK
        string content
        string comments_json
        datetime created_at
    }

    COMMENT {
        string comment_id PK
        string post_id FK
        string parent_comment_id FK
        string user_id FK
        string content
        datetime created_at
    }

    BACKFILL_CHECKPOINT {
        string checkpoint_id PK
        string last_post_id_processed
        string status
        datetime updated_at
    }

    ORDER_CONSISTENCY_CHECK {
        string check_id PK
        string post_id FK
        boolean order_matches
        datetime checked_at
    }
```
