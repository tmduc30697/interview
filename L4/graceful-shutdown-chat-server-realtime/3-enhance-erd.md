# Enhance ERD — Thêm hàng đợi bền vững, presence có TTL, kế hoạch giới hạn reconnect

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- `MESSAGE` có thêm trạng thái `queued_durable` và cột `delivered_at`/`acked_by_client_at` — tin nhắn chưa được client ACK nay được ghi vào store bền vững thay vì chỉ giữ trong memory, và server theo dõi được đã gửi/đã ACK để tránh gửi trùng khi reconnect — đáp ứng yêu cầu 1 và yêu cầu 3.
- `CONNECTION` có thêm trạng thái `draining` và cột `close_code` — đáp ứng yêu cầu 2 (mã lý do riêng cho "server draining").
- Entity mới `PRESENCE_STATE` — đại diện trạng thái tạm thời như đang gõ hoặc đang trong cuộc gọi, có `expires_at` để không bao giờ treo vĩnh viễn — đáp ứng yêu cầu 4.
- Entity mới `CLUSTER_DRAIN_PLAN` — giới hạn tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    USER ||--o{ CONNECTION : "opens"
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ PRESENCE_STATE : "has"
    CLUSTER_DRAIN_PLAN ||--o{ CONNECTION : coordinates

    USER {
        string id PK
        string name
    }
    CONVERSATION {
        string id PK
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string content
        string status "pending | queued_durable | delivered | acked"
        datetime delivered_at
        datetime acked_by_client_at
    }
    CONNECTION {
        string id PK
        string user_id FK
        string server_instance_id
        string status "connected | draining | closed"
        int close_code "vd 4000=draining, khac voi 1006=loi bat thuong"
        datetime connected_at
    }
    PRESENCE_STATE {
        string id PK
        string conversation_id FK
        string user_id FK
        string type "typing | in_call"
        datetime started_at
        datetime expires_at
    }
    CLUSTER_DRAIN_PLAN {
        string id PK
        string deploy_id
        int total_online_users
        float max_percent_reconnect_simultaneous
        int current_batch_reconnecting
    }
```
