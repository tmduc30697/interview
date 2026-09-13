# Enhance ERD — Token bucket theo scope, policy theo trust level, phát hiện pattern nội dung

Đây là **enhance**, mô hình dữ liệu sau khi áp thuật toán rate limit cho phép burst + phân biệt theo cuộc trò chuyện + phát hiện pattern nội dung + policy động theo uy tín tài khoản lên base. So với base: `RATE_LIMIT_COUNTER` (cửa sổ cố định) được thay bằng `RATE_LIMIT_BUCKET` (token bucket cho phép burst ngắn, tách theo `scope` — trong 1 conversation cụ thể hay tổng hợp toàn user). `USER` thêm `trust_level` (suy ra từ tuổi tài khoản + lịch sử hành vi), gắn với `RATE_LIMIT_POLICY` quy định `burst_capacity`/`sustained_rate` khác nhau theo `trust_level` và loại conversation. Thêm mới `CONTENT_SPAM_SIGNAL` — theo dõi nội dung lặp lại gửi tới nhiều đối tượng khác nhau trong thời gian ngắn, phát hiện bot dù tần suất từng tin không vượt ngưỡng cứng.

```mermaid
erDiagram
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ CONVERSATION_MEMBER : has
    USER ||--o{ CONVERSATION_MEMBER : "is member of"
    USER ||--o{ RATE_LIMIT_BUCKET : "throttled by"
    USER ||--|| RATE_LIMIT_POLICY : "governed by (theo trust_level hiện tại)"
    USER ||--o{ CONTENT_SPAM_SIGNAL : "flagged by"

    USER {
        string id PK
        string name
        datetime created_at
        boolean is_verified
        string trust_level "new | standard | trusted"
    }
    CONVERSATION {
        string id PK
        string type "direct | group"
        int member_count
    }
    CONVERSATION_MEMBER {
        string conversation_id FK
        string user_id FK
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string content
        string content_hash "dùng để so khớp nội dung lặp lại"
        datetime sent_at
    }
    RATE_LIMIT_BUCKET {
        string id PK
        string user_id FK
        string scope "per_conversation:<id> | per_user_aggregate"
        decimal tokens_remaining
        decimal bucket_capacity "burst tối đa cho phép"
        decimal refill_rate_per_sec "tốc độ nạp lại token, quyết định tần suất bền vững"
        datetime last_refill_at
    }
    RATE_LIMIT_POLICY {
        string id PK
        string trust_level "new | standard | trusted"
        string conversation_type "direct | group"
        decimal burst_capacity
        decimal sustained_rate_per_sec
    }
    CONTENT_SPAM_SIGNAL {
        string id PK
        string user_id FK
        string content_hash
        int distinct_target_count "số conversation/user khác nhau nhận cùng nội dung"
        datetime first_seen_at
        datetime last_seen_at
        string pattern_detected "regular_interval_bot | none"
    }
```
