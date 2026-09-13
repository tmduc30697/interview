# Enhance ERD — sau khi có virtual waiting room + admission control

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `USER` bổ sung `tier` — cơ sở cho chính sách ưu tiên công bố trước (VIP) khi hàng chờ đông.
- `WAITING_ROOM_QUEUE_ENTRY` (mới) — vị trí, thời điểm vào hàng, trạng thái chờ/đã vào/hết hạn.
- `ADMISSION_TOKEN` (mới) — cấp cho user đã qua hàng chờ, dùng để checkout mà không bị throttle lại giữa đường.
- `ADMISSION_RATE_CONFIG` + `BACKEND_HEALTH_SNAPSHOT` (mới) — tốc độ cho qua hiện tại được điều chỉnh động theo sức khỏe backend thực tế, không dùng số tĩnh.
- `THROUGHPUT_METRIC` (mới) — throughput checkout thành công/giây và tỉ lệ user phải chờ, phục vụ lên kế hoạch capacity.

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    PRODUCT ||--o{ ORDER : "ordered as"
    USER ||--o{ WAITING_ROOM_QUEUE_ENTRY : joins
    WAITING_ROOM_QUEUE_ENTRY ||--o| ADMISSION_TOKEN : grants
    ADMISSION_TOKEN ||--o| ORDER : authorizes
    BACKEND_HEALTH_SNAPSHOT ||--o{ ADMISSION_RATE_CONFIG : informs

    USER {
        string id PK
        string name
        string tier "regular | vip"
    }
    PRODUCT {
        string id PK
        string name
        int flash_sale_stock
        decimal price
    }
    ORDER {
        string id PK
        string user_id FK
        string product_id FK
        string status "pending | paid | failed"
        datetime created_at
    }
    WAITING_ROOM_QUEUE_ENTRY {
        string id PK
        string user_id FK
        string priority_tier "vip | regular"
        int position
        datetime joined_at
        string status "waiting | admitted | expired"
    }
    ADMISSION_TOKEN {
        string id PK
        string queue_entry_id FK
        string user_id FK
        datetime issued_at
        datetime expires_at
        boolean used_for_checkout
    }
    ADMISSION_RATE_CONFIG {
        string id PK
        int current_admit_rate_per_sec
        string reason "healthy | db_latency_high | error_rate_high"
        datetime updated_at
    }
    BACKEND_HEALTH_SNAPSHOT {
        string id PK
        datetime measured_at
        int db_latency_ms
        decimal error_rate_pct
        string status "healthy | degraded"
    }
    THROUGHPUT_METRIC {
        string id PK
        datetime window_start
        datetime window_end
        decimal successful_checkouts_per_sec
        int total_user_count
        int queued_user_count
        decimal queued_ratio_pct
    }
```
