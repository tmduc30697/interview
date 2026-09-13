# Base ERD — Marketplace với session "remember me" nhưng chưa tách biệt hành động nhạy cảm

Đây là **base**: trạng thái hệ thống marketplace *trước khi* có theo dõi bất thường và step-up cho hành động nhạy cảm. Suy luận từ đề bài, base đã có `USER` với `SESSION` dạng "remember me" (đăng nhập 1 lần, giữ lâu dài), `ADDRESS` và `PAYMENT_METHOD` được lưu sẵn để mua nhanh, và `ORDER` khi thanh toán. Base coi session còn hạn là đủ điều kiện cho mọi hành động — kể cả đổi địa chỉ, đổi phương thức thanh toán, hay thanh toán — không có bước xác thực bổ sung nào, cũng chưa theo dõi tín hiệu bất thường trong phiên.

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ ADDRESS : has
    USER ||--o{ PAYMENT_METHOD : has
    USER ||--o{ ORDER : places
    ADDRESS ||--o{ ORDER : "giao tới"
    PAYMENT_METHOD ||--o{ ORDER : "thanh toán bằng"

    USER {
        string id PK
        string email
    }
    SESSION {
        string id PK
        string user_id FK
        boolean remember_me
        string device_name
        string ip_address
        string user_agent
        datetime created_at
        datetime expires_at
    }
    ADDRESS {
        string id PK
        string user_id FK
        string address_line
        boolean is_default
        datetime updated_at
    }
    PAYMENT_METHOD {
        string id PK
        string user_id FK
        string type
        string last4
        boolean is_default
        datetime updated_at
    }
    ORDER {
        string id PK
        string user_id FK
        string address_id FK
        string payment_method_id FK
        decimal amount
        string status "pending|paid|failed"
        datetime created_at
    }
```
