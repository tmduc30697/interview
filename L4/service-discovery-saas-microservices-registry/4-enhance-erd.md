# ERD — Enhance (sau khi có health check/cache/TTL đầy đủ)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — metadata version/zone khi đăng ký, active/passive health check log, TTL/heartbeat để phát hiện crash, và cache phía client với TTL/invalidate. So với base, `INSTANCE` nay có `version`, `zone`, `health_check_mode` và `ttl_expires_at` để registry tự loại instance chết mà không cần chờ deregister.

```mermaid
erDiagram
    SERVICE ||--o{ INSTANCE : "has"
    INSTANCE ||--o{ HEALTH_CHECK_LOG : "checked by"
    SERVICE ||--o{ CLIENT_CACHE_ENTRY : "cached as"

    SERVICE {
        string service_id PK
        string name
    }

    INSTANCE {
        string instance_id PK
        string service_id FK
        string host
        int port
        string version
        string zone
        string status
        string health_check_mode
        datetime registered_at
        datetime last_heartbeat_at
        datetime ttl_expires_at
    }

    HEALTH_CHECK_LOG {
        string check_id PK
        string instance_id FK
        string mode
        string result
        datetime checked_at
    }

    CLIENT_CACHE_ENTRY {
        string cache_id PK
        string service_id FK
        string caller_name
        datetime cached_at
        datetime ttl_expires_at
    }
```
