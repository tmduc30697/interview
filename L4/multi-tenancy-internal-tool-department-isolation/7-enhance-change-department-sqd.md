# Enhance sequence — Chuyển phòng ban có hiệu lực theo thời điểm

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ có `EMPLOYEE.department_id` tĩnh, chưa có quy trình chuyển đổi có kiểm soát thời điểm hiệu lực. Đáp ứng yêu cầu 2 của đề bài: quyền truy cập dữ liệu phòng ban cũ bị thu hồi đúng lúc chuyển đổi có hiệu lực, còn tài liệu nhân viên đã tạo khi ở phòng cũ vẫn giữ nguyên `department_id` cũ.

```mermaid
sequenceDiagram
    actor HR as Nhân sự / Quản lý
    participant App as HR Service
    participant Transfer as DEPARTMENT_TRANSFER
    participant Scheduler as Scheduled Job (theo effective_at)
    participant Emp as EMPLOYEE store
    participant DB as DOCUMENT store

    HR->>App: Chuyển nhân viên An từ phòng kỹ thuật sang phòng kinh doanh, hiệu lực từ đầu tháng sau
    App->>Transfer: Tạo DEPARTMENT_TRANSFER(from=kỹ thuật, to=kinh doanh, effective_at=đầu tháng sau)
    Note over Emp: Trước effective_at, employee.department_id vẫn là kỹ thuật — quyền truy cập dữ liệu phòng kỹ thuật vẫn còn nguyên

    Scheduler->>Transfer: Quét các DEPARTMENT_TRANSFER có effective_at <= now và chưa áp dụng
    Transfer-->>Scheduler: Bản ghi transfer của An đã tới hạn
    Scheduler->>Emp: Cập nhật EMPLOYEE(An).department_id = kinh doanh
    Note over Emp: Từ thời điểm này, các truy vấn xem/tìm kiếm tài liệu department_only của phòng kỹ thuật sẽ tự động từ chối An do department_id không còn khớp

    App->>DB: Kiểm tra DOCUMENT do An tạo khi còn ở phòng kỹ thuật
    DB-->>App: department_id của các tài liệu này vẫn là kỹ thuật, không đổi
    Note over DB: Tài liệu/phê duyệt cũ vẫn thuộc phòng kỹ thuật cho mục đích lưu trữ, không tự động chuyển theo An sang phòng kinh doanh
```
