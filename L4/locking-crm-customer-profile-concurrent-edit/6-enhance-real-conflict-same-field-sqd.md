# Enhance sequence — Xung đột thật khi cùng sửa 1 trường trong cùng nhóm

Đây là **enhance**, flow mới đáp ứng yêu cầu 2 của đề bài: cả sale và CSKH cùng sửa số điện thoại liên hệ chính (cùng thuộc `CUSTOMER_GENERAL_INFO`) gần như đồng thời. Vì cùng 1 nhóm trường, optimistic lock ở đúng cấp nhóm phải từ chối người lưu sau, hiển thị đúng giá trị mới nhất, không tự động merge 2 số điện thoại thành 1 giá trị mơ hồ — cũng minh hoạ yêu cầu 4 (chỉ rõ đúng trường/nhóm bị đổi).

```mermaid
sequenceDiagram
    actor Sale
    actor CSKH
    participant CRM as CRM Service
    participant GeneralDB as CUSTOMER_GENERAL_INFO

    Sale->>CRM: Mở hồ sơ C, nhận general_info version = 3 (phone cũ sai)
    CSKH->>CRM: Mở cùng hồ sơ C, cũng nhận general_info version = 3

    Sale->>CRM: Sửa phone thành số mới A, gửi version = 3
    CRM->>GeneralDB: UPDATE CUSTOMER_GENERAL_INFO SET phone = A, version = 4 WHERE customer_id = C AND version = 3
    GeneralDB-->>CRM: 1 row affected, thành công
    CRM-->>Sale: Lưu thành công, general_info version mới = 4

    CSKH->>CRM: Sửa phone thành số mới B, gửi version = 3 (đã lỗi thời)
    CRM->>GeneralDB: UPDATE CUSTOMER_GENERAL_INFO SET phone = B, version = 4 WHERE customer_id = C AND version = 3
    GeneralDB-->>CRM: 0 row affected (version hiện tại đã là 4)
    CRM->>GeneralDB: SELECT phone, version FROM CUSTOMER_GENERAL_INFO WHERE customer_id = C
    GeneralDB-->>CRM: phone = A, version = 4
    CRM-->>CSKH: Từ chối lưu, báo rõ "trường phone trong nhóm thông tin chung vừa được Sale cập nhật thành A", không tự merge A và B
    Note over CRM,CSKH: CSKH thấy đúng trường nào xung đột (phone), không bị báo cả hồ sơ đã cũ, chỉ cần xác nhận lại đúng phần này
```
