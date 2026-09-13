# Base sequence — Update config (optimistic lock ở cấp cả nhóm, thành công)

Đây là **base**, flow lưu thay đổi config hiện tại: optimistic lock áp dụng cho toàn bộ `CONFIG_GROUP`, chỉ 1 cột `version` chung cho mọi config/flag trong nhóm. Khi chỉ có 1 kỹ sư sửa tại 1 thời điểm, flow hoạt động bình thường — đây là tiền đề để so sánh với flow tiếp theo khi 2 kỹ sư sửa 2 config không liên quan trong cùng nhóm.

```mermaid
sequenceDiagram
    actor EngineerB as Kỹ sư B
    participant Dash as Config Dashboard
    participant DB as CONFIG_GROUP table

    EngineerB->>Dash: Mở nhóm config, nhận version = 12
    EngineerB->>Dash: Sửa 1 config không liên quan tới flag khẩn cấp, gửi kèm version = 12
    Dash->>DB: UPDATE CONFIG_GROUP SET ..., version = 13 WHERE id = G AND version = 12
    DB-->>Dash: 1 row affected, thành công
    Dash-->>EngineerB: Lưu thành công, version mới = 13
    Note over Dash,DB: Vì chỉ 1 người sửa tại 1 thời điểm, version chung chưa gây vấn đề gì
```
