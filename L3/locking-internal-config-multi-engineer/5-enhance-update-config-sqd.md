# Enhance sequence — Update config (conflict theo đúng item, hiển thị diff 3 bên)

Đây là **enhance** của flow `update-config` đã có ở base. So với base (chỉ báo chung chung "đã được cập nhật"), khi version của đúng `CONFIG_ITEM` đang sửa bị lệch, hệ thống phải hiển thị rõ diff giữa bản kỹ sư A đang muốn lưu, bản gốc A đã đọc, và bản mới nhất kỹ sư B đã lưu — đáp ứng đúng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A
    actor EngineerB as Kỹ sư B
    participant Dash as Config Dashboard
    participant DB as CONFIG_ITEM table
    participant Log as CONFIG_AUDIT_LOG

    EngineerA->>Dash: Mở config "max_retry_count", nhận value=3, version=5
    EngineerB->>Dash: Mở cùng config "max_retry_count", cũng nhận value=3, version=5

    EngineerB->>Dash: Lưu value=5, gửi version=5
    Dash->>DB: UPDATE CONFIG_ITEM SET value=5, version=6 WHERE id=X AND version=5
    DB-->>Dash: 1 row affected, thành công
    Dash->>Log: Ghi CONFIG_AUDIT_LOG (old_value=3, new_value=5, changed_by=B, is_emergency_override=false)
    Dash-->>EngineerB: Lưu thành công, version mới = 6

    EngineerA->>Dash: Lưu value=10, vẫn gửi version=5 (đã lỗi thời)
    Dash->>DB: UPDATE CONFIG_ITEM SET value=10, version=6 WHERE id=X AND version=5
    DB-->>Dash: 0 row affected (version hiện tại đã là 6)
    Dash->>DB: SELECT value, version FROM CONFIG_ITEM WHERE id=X
    DB-->>Dash: value=5, version=6
    Dash-->>EngineerA: Từ chối lưu, hiển thị diff 3 bên, bản A muốn lưu (10), bản gốc A đã đọc (3), bản mới nhất B đã lưu (5)
    Note over EngineerA,Dash: A tự quyết định merge tay chính xác dựa trên 3 giá trị này, hệ thống không tự động merge ngầm với config nhạy cảm production
```
