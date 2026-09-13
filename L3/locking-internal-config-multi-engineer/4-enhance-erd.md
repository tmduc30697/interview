# Enhance ERD — Version theo từng config item, đường khẩn cấp, audit log immutable, notify real-time

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (`version` chung ở `CONFIG_GROUP`), thay đổi và bổ sung:

- `version` chuyển xuống cấp `CONFIG_ITEM` (mỗi flag/config riêng lẻ có version riêng) — đáp ứng yêu cầu 1 (conflict theo đúng item) và yêu cầu 2 (A tắt flag X không còn bị chặn bởi thay đổi không liên quan của B).
- `CONFIG_ITEM` thêm `is_emergency_override`/liên kết tới log để phân biệt lần lưu nào đã bấm "buộc lưu" — đáp ứng yêu cầu 3.
- Entity mới `CONFIG_AUDIT_LOG` (immutable, append-only, không có API xóa/sửa) — ghi mọi thay đổi kể cả qua đường khẩn cấp, đáp ứng yêu cầu 4.
- Entity mới `CONFIG_EDIT_SESSION` — theo dõi kỹ sư nào đang mở màn hình sửa config nào, làm nền cho việc đẩy thông báo real-time qua WebSocket/polling, đáp ứng yêu cầu 5.

```mermaid
erDiagram
    ENGINEER ||--o{ CONFIG_ITEM : "cập nhật gần nhất"
    CONFIG_GROUP ||--o{ CONFIG_ITEM : "chứa"
    CONFIG_ITEM ||--o{ CONFIG_AUDIT_LOG : "có lịch sử thay đổi"
    CONFIG_ITEM ||--o{ CONFIG_EDIT_SESSION : "đang được xem/sửa bởi"
    ENGINEER ||--o{ CONFIG_EDIT_SESSION : "mở phiên xem/sửa"
    ENGINEER ||--o{ CONFIG_AUDIT_LOG : "thực hiện thay đổi"

    ENGINEER {
        string id PK
        string name
        string team
    }
    CONFIG_GROUP {
        string id PK
        string name "vd checkout-service-config"
    }
    CONFIG_ITEM {
        string id PK
        string group_id FK
        string key "vd feature_flag_x"
        string value
        string type "feature_flag|config"
        int version "version riêng cho từng item"
        string updated_by FK
        datetime updated_at
    }
    CONFIG_AUDIT_LOG {
        string id PK
        string config_item_id FK
        string changed_by FK
        datetime changed_at
        string old_value
        string new_value
        bool is_emergency_override "true nếu dùng đường buộc lưu bỏ qua version check"
        string note
    }
    CONFIG_EDIT_SESSION {
        string id PK
        string config_item_id FK
        string engineer_id FK
        int last_seen_version "version engineer này đang nhìn thấy trên UI"
        datetime connected_at
    }
```
