# Base ERD — core-api trước khi có blue-green deployment

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có blue-green deployment. Đề bài nói service "core-api" được deploy nhiều lần/tuần, có migration schema DB đi kèm, và có session/kết nối dài (WebSocket) đang mở — nên base cần đủ: service, deployment (chỉ 1 target đang chạy tại 1 thời điểm), router trỏ thẳng tới target đó, migration DB, và các connection đang mở trên deployment. Chưa có entity nào phục vụ smoke test/switch có thể đảo ngược/rollback policy tự động/connection draining — những thứ đó là phần enhance.

```mermaid
erDiagram
    SERVICE ||--o{ DEPLOYMENT : has
    SERVICE ||--|| ROUTER_CONFIG : "routes via (1 target)"
    SERVICE ||--o{ DB_MIGRATION : applies
    DEPLOYMENT ||--o{ CLIENT_CONNECTION : serves

    SERVICE {
        string id PK
        string name "core-api"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string status "active"
        datetime deployed_at
    }
    ROUTER_CONFIG {
        string id PK
        string service_id FK
        string target_deployment_id FK
    }
    DB_MIGRATION {
        string id PK
        string service_id FK
        string version
        string description
        datetime applied_at
    }
    CLIENT_CONNECTION {
        string id PK
        string deployment_id FK
        string type "websocket | long_request"
        datetime opened_at
        string status
    }
```
