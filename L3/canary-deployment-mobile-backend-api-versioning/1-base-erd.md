# Base ERD — Canary backend mobile trước khi nhận diện đa phiên bản app

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** canary nhận diện đa phiên bản app client. Đề bài nói backend đã phục vụ nhiều phiên bản app cũ và có sẵn khái niệm canary — nên base cần đủ: phiên bản app client, backend deployment (đã có role stable/canary), test hợp đồng API (chỉ test với app mới nhất), và log request (có ghi app_client_version_header nhưng chưa dùng để tách metric). Chưa có entity nào phục vụ stratify traffic theo version/metric tách theo version/kế hoạch rollback tương thích ngược — những thứ đó là phần enhance.

```mermaid
erDiagram
    BACKEND_DEPLOYMENT ||--o{ API_CONTRACT_TEST : "tested by"
    BACKEND_DEPLOYMENT ||--o{ API_REQUEST_LOG : serves
    BACKEND_DEPLOYMENT ||--o{ ERROR_METRIC_WINDOW : measured

    APP_CLIENT_VERSION {
        string id PK
        string version
        int active_user_count
    }
    BACKEND_DEPLOYMENT {
        string id PK
        string version
        string role "stable | canary"
        int traffic_percent
    }
    API_CONTRACT_TEST {
        string id PK
        string backend_version FK
        string tested_against_app_version
        string result
    }
    API_REQUEST_LOG {
        string id PK
        string backend_version FK
        string app_client_version_header
        int status_code
        boolean parse_error
        datetime created_at
    }
    ERROR_METRIC_WINDOW {
        string id PK
        string backend_version FK
        datetime window_start
        datetime window_end
        decimal error_rate "gộp chung mọi client version"
    }
```
