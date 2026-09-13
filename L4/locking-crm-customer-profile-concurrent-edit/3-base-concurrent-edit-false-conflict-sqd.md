# Base sequence — Xung đột giả khi 2 người sửa 2 nhóm trường khác nhau

Đây là **base**, mô tả đúng kịch bản nêu ở yêu cầu 1 của đề bài: nhân viên sale mở hồ sơ C ở version 8, cập nhật giai đoạn deal; gần như cùng lúc nhân viên CSKH cũng mở hồ sơ C ở version 8, thêm ghi chú cuộc gọi. Vì optimistic lock ở cấp toàn bộ record, ai lưu sau sẽ bị từ chối dù 2 người sửa 2 nhóm trường hoàn toàn khác nhau — đây chính là vấn đề mà việc tách lock theo nhóm trường (enhance) phải giải quyết.

```mermaid
sequenceDiagram
    actor Sale
    actor CSKH
    participant CRM as CRM Service
    participant DB as CUSTOMER_PROFILE table

    Sale->>CRM: Mở hồ sơ khách hàng C, nhận version = 8
    CSKH->>CRM: Mở cùng hồ sơ khách hàng C, cũng nhận version = 8

    Sale->>CRM: Cập nhật deal_stage "đang đàm phán" -> "chốt hợp đồng", gửi version = 8
    CRM->>DB: UPDATE CUSTOMER_PROFILE SET deal_stage = ..., version = 9 WHERE id = C AND version = 8
    DB-->>CRM: 1 row affected, thành công
    CRM-->>Sale: Lưu thành công, version mới = 9

    CSKH->>CRM: Thêm ghi chú cuộc gọi vừa thực hiện, gửi version = 8 (đã lỗi thời)
    CRM->>DB: UPDATE CUSTOMER_PROFILE SET contact_notes = ..., version = 9 WHERE id = C AND version = 8
    DB-->>CRM: 0 row affected (version hiện tại đã là 9 do Sale vừa lưu)
    CRM-->>CSKH: Từ chối lưu, báo "hồ sơ đã được người khác cập nhật"
    Note over CSKH,CRM: CSKH bị từ chối dù ghi chú cuộc gọi hoàn toàn không liên quan tới deal_stage mà Sale vừa sửa, đây là xung đột giả do version chung toàn record
```
