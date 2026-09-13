# Enhance sequence — Pessimistic lock khi chuyển giao khách hàng

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 3 của đề bài: khi chuyển giao khách hàng từ sale này sang sale khác, cần khóa cứng toàn bộ hồ sơ trong vài giây để không ai (kể cả CSKH) sửa bất kỳ trường nào giữa chừng — cơ chế khóa tường minh có timeout ngắn, tách biệt hẳn với optimistic lock mặc định dùng cho sửa thông thường.

```mermaid
sequenceDiagram
    actor SaleOld as Sale cũ
    actor CSKH
    participant CRM as CRM Service
    participant LockStore as CUSTOMER_HANDOVER_LOCK
    participant GeneralDB as CUSTOMER_GENERAL_INFO
    participant SaleDB as CUSTOMER_SALE_INFO

    SaleOld->>CRM: Bắt đầu chuyển giao khách hàng C sang Sale mới
    CRM->>LockStore: Acquire CUSTOMER_HANDOVER_LOCK (customer_id=C, locked_by=SaleOld, TTL=5s)
    LockStore-->>CRM: Lock granted, lock_token=T1

    CSKH->>CRM: Trong lúc đó, cố gắng cập nhật contact_notes của hồ sơ C
    CRM->>LockStore: Kiểm tra có handover lock active cho customer_id=C không
    LockStore-->>CRM: Có, đang bị khóa bởi SaleOld
    CRM-->>CSKH: Từ chối, báo "hồ sơ đang trong quá trình chuyển giao, thử lại sau vài giây"

    CRM->>SaleDB: Cập nhật owner mới cho toàn bộ CUSTOMER_SALE_INFO
    CRM->>GeneralDB: Cập nhật thông tin liên quan chuyển giao ở CUSTOMER_GENERAL_INFO nếu cần
    CRM->>LockStore: Release CUSTOMER_HANDOVER_LOCK (lock_token=T1)
    LockStore-->>CRM: Released
    CRM-->>SaleOld: Chuyển giao hoàn tất
    Note over LockStore,CSKH: Sau khi lock được release (hoặc tự hết TTL nếu quá trình treo), các thao tác sửa bình thường qua optimistic lock mới được phép tiếp tục
```
