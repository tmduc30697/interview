# Enhance sequence — Update profile (optimistic lock riêng theo nhóm trường, hết xung đột giả)

Đây là **enhance** của flow `update-profile`, áp dụng đúng lên kịch bản đã gây xung đột giả ở base (`concurrent-edit-false-conflict`). So với base (1 version chung), giờ Sale sửa `CUSTOMER_SALE_INFO` và CSKH sửa `CUSTOMER_CSKH_INFO` dùng 2 version độc lập, nên cả 2 đều lưu thành công. Đáp ứng trực tiếp yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor Sale
    actor CSKH
    participant CRM as CRM Service
    participant SaleDB as CUSTOMER_SALE_INFO
    participant CskhDB as CUSTOMER_CSKH_INFO
    participant Log as CUSTOMER_FIELD_CHANGE_LOG

    Sale->>CRM: Mở hồ sơ khách hàng C, nhận sale_info version = 4
    CSKH->>CRM: Mở cùng hồ sơ khách hàng C, nhận cskh_info version = 6

    Sale->>CRM: Cập nhật deal_stage, gửi kèm sale_info version = 4
    CRM->>SaleDB: UPDATE CUSTOMER_SALE_INFO SET deal_stage = ..., version = 5 WHERE customer_id = C AND version = 4
    SaleDB-->>CRM: 1 row affected, thành công
    CRM->>Log: Ghi CUSTOMER_FIELD_CHANGE_LOG (field_group=sale, field_name=deal_stage, old/new value)
    CRM-->>Sale: Lưu thành công, sale_info version mới = 5

    CSKH->>CRM: Thêm ghi chú cuộc gọi, gửi kèm cskh_info version = 6
    CRM->>CskhDB: UPDATE CUSTOMER_CSKH_INFO SET contact_notes = ..., version = 7 WHERE customer_id = C AND version = 6
    CskhDB-->>CRM: 1 row affected, thành công (không liên quan gì tới version của sale_info)
    CRM->>Log: Ghi CUSTOMER_FIELD_CHANGE_LOG (field_group=cskh, field_name=contact_notes, old/new value)
    CRM-->>CSKH: Lưu thành công, cskh_info version mới = 7
    Note over SaleDB,CskhDB: Cả 2 đều lưu thành công vì mỗi nhóm trường có version riêng, không còn xung đột giả
```
