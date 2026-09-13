# Enhance sequence — Tra cứu lịch sử thay đổi từng trường để xử lý khiếu nại

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 5 của đề bài: mọi thay đổi hồ sơ khách hàng (theo từng nhóm trường) được ghi lịch sử đầy đủ ai sửa, khi nào, giá trị trước/sau, phục vụ tra cứu khi có tranh chấp nội bộ về việc ai đã cập nhật sai thông tin.

```mermaid
sequenceDiagram
    actor Manager as Quản lý xử lý khiếu nại
    participant CRM as CRM Service
    participant Log as CUSTOMER_FIELD_CHANGE_LOG

    Manager->>CRM: Tra cứu lịch sử thay đổi trường phone của khách hàng C
    CRM->>Log: SELECT * FROM CUSTOMER_FIELD_CHANGE_LOG WHERE customer_id = C AND field_name = 'phone' ORDER BY changed_at
    Log-->>CRM: Danh sách thay đổi: Sale sửa phone từ giá trị cũ sang A lúc 10:00, CSKH cố sửa sang B bị từ chối lúc 10:01
    CRM-->>Manager: Hiển thị timeline đầy đủ ai sửa, khi nào, giá trị trước/sau cho trường phone

    Manager->>CRM: Tra cứu rộng hơn toàn bộ thay đổi nhóm sale_info trong tuần qua
    CRM->>Log: SELECT * FROM CUSTOMER_FIELD_CHANGE_LOG WHERE customer_id = C AND field_group = 'sale' AND changed_at >= ...
    Log-->>CRM: Danh sách đầy đủ thay đổi deal_stage kèm ai sửa
    CRM-->>Manager: Manager xác định chính xác ai đã cập nhật sai thông tin dẫn tới hậu quả nghiệp vụ
```
