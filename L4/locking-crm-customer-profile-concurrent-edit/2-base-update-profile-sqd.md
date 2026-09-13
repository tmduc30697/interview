# Base sequence — Update profile (optimistic lock ở cấp toàn bộ record, thành công)

Đây là **base**, flow cập nhật hồ sơ khách hàng ở trạng thái hiện tại: optimistic lock áp dụng cho toàn bộ `CUSTOMER_PROFILE`, chỉ 1 cột `version` chung. Khi chỉ có 1 người sửa tại 1 thời điểm, flow hoạt động bình thường — đây là tiền đề để so sánh với flow tiếp theo khi 2 người sửa 2 nhóm trường khác nhau cùng lúc.

```mermaid
sequenceDiagram
    actor Sale
    participant CRM as CRM Service
    participant DB as CUSTOMER_PROFILE table

    Sale->>CRM: Mở hồ sơ khách hàng C, nhận version = 8
    Sale->>CRM: Cập nhật deal_stage, gửi kèm version = 8
    CRM->>DB: UPDATE CUSTOMER_PROFILE SET deal_stage = ..., version = 9 WHERE id = C AND version = 8
    DB-->>CRM: 1 row affected, thành công
    CRM-->>Sale: Lưu thành công, version mới = 9
    Note over CRM,DB: Vì chỉ 1 người sửa tại 1 thời điểm, version chung không gây vấn đề gì
```
