# Base ERD — Chat/group với rate limit ngưỡng cứng cố định

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có thuật toán cho phép burst tự nhiên, phân biệt theo cuộc trò chuyện, phát hiện pattern nội dung, và điều chỉnh theo uy tín tài khoản. Đề bài nói tới ngưỡng rate limit cứng (giới hạn tin/giây cố định) đang gây chặn nhầm user thật — nên base cần đủ: user, cuộc trò chuyện (1-1 hoặc group), tin nhắn, và 1 bộ đếm rate limit đơn giản theo cửa sổ thời gian cố định áp dụng chung cho mọi user, không phân biệt loại cuộc trò chuyện hay uy tín tài khoản. Base **chưa có** token bucket cho phép burst, chưa có policy theo trust level, chưa có phát hiện pattern nội dung lặp lại — những thứ đó là phần enhance.

```mermaid
erDiagram
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ CONVERSATION_MEMBER : has
    USER ||--o{ CONVERSATION_MEMBER : "is member of"
    USER ||--o{ RATE_LIMIT_COUNTER : "tracked by"

    USER {
        string id PK
        string name
        datetime created_at
        boolean is_verified
    }
    CONVERSATION {
        string id PK
        string type "direct | group"
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
        datetime sent_at
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string user_id FK
        datetime window_start
        int message_count "reset mỗi cửa sổ cố định, vd mỗi giây"
    }
```
