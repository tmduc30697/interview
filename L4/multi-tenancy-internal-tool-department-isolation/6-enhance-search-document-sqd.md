# Enhance sequence — Tìm kiếm tài liệu (kết quả lọc theo phạm vi truy cập, không lộ tồn tại)

Đây là **enhance**, cùng flow "Tìm kiếm tài liệu" đã có ở base nhưng nay thay đổi theo yêu cầu 4 của đề bài: trước khi trả kết quả (kể cả tiêu đề và số lượng), hệ thống lọc chỉ giữ lại tài liệu nằm trong phạm vi nhìn thấy được của người tìm kiếm — department_only của chính phòng ban mình, company_wide, và cross_department_shared còn hiệu lực — nên tài liệu ngoài phạm vi không xuất hiện dưới bất kỳ hình thức nào, kể cả gợi ý autocomplete.

```mermaid
sequenceDiagram
    actor Emp as Nhân viên (phòng kinh doanh)
    participant App as Search Service
    participant Index as Search Index (toàn bộ DOCUMENT)
    participant Share as CROSS_DEPARTMENT_SHARE

    Emp->>App: Gõ từ khóa "tái cơ cấu"
    App->>App: Xác định phạm vi nhìn thấy = department_id hiện tại + company_wide + cross_department_shared còn hiệu lực
    App->>Share: Lấy danh sách document_id đang chia sẻ hợp lệ với phòng kinh doanh
    Share-->>App: Danh sách document_id còn hiệu lực (rỗng trong trường hợp này)
    App->>Index: Query full-text, áp filter resource_type + department_id/scope trước khi chấm điểm kết quả
    Index-->>App: Chỉ trả các tài liệu nằm trong phạm vi nhìn thấy của Emp
    App-->>Emp: Hiển thị tiêu đề + số lượng kết quả, không bao gồm "Kế hoạch tái cơ cấu phòng kỹ thuật"
    Note over App,Index: Tài liệu ngoài phạm vi bị loại khỏi index ngay ở bước filter, không chỉ ẩn nội dung — nên không lộ cả sự tồn tại qua autocomplete hay đếm số lượng kết quả
```
