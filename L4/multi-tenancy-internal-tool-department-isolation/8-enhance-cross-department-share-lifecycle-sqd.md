# Enhance sequence — Vòng đời chia sẻ liên phòng ban (tạo có thời hạn, tự động thu hồi)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có khái niệm chia sẻ liên phòng ban, mọi tài liệu chỉ thuộc đúng 1 phòng. Đáp ứng yêu cầu 1 và 3 của đề bài: dự án liên phòng ban chia sẻ có chọn lọc một phần dữ liệu trong khoảng thời gian nhất định, và quyền này tự động thu hồi khi hết hạn thay vì phải nhớ thu hồi thủ công.

```mermaid
sequenceDiagram
    actor Owner as Nhân viên phòng kỹ thuật (chủ tài liệu)
    participant App as Internal Tool Service
    participant DB as DOCUMENT store
    participant Share as CROSS_DEPARTMENT_SHARE
    participant Target as SHARE_TARGET_DEPARTMENT
    participant Scheduler as Scheduled Job (theo valid_until)
    actor Biz as Nhân viên phòng kinh doanh (dự án liên phòng ban)

    Owner->>App: Chia sẻ tài liệu dự án X cho phòng kinh doanh, từ nay đến hết quý
    App->>DB: Cập nhật DOCUMENT(X).resource_type = cross_department_shared
    App->>Share: Tạo CROSS_DEPARTMENT_SHARE(document_id=X, valid_from=now, valid_until=cuối quý, status=active)
    App->>Target: Tạo SHARE_TARGET_DEPARTMENT(share_id, department_id=kinh doanh)

    Biz->>App: Xem tài liệu dự án X (trong thời hạn chia sẻ)
    App->>Share: Kiểm tra active, now trong [valid_from, valid_until], department khớp
    Share-->>App: Hợp lệ
    App-->>Biz: Cho phép xem nội dung

    Scheduler->>Share: Quét các CROSS_DEPARTMENT_SHARE có valid_until <= now và status=active
    Share-->>Scheduler: Bản ghi chia sẻ tài liệu X đã hết hạn
    Scheduler->>Share: Cập nhật status=expired
    Note over Share: Không cần ai nhớ thu hồi thủ công, hết hạn là tự động mất quyền

    Biz->>App: Thử xem lại tài liệu dự án X sau khi hết hạn
    App->>Share: Kiểm tra CROSS_DEPARTMENT_SHARE, status=expired
    App-->>Biz: Từ chối truy cập, chia sẻ đã hết hiệu lực
```
