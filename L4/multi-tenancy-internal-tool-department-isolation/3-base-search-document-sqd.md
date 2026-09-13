# Base sequence — Tìm kiếm tài liệu (chưa lọc theo phòng ban, lộ sự tồn tại)

Đây là **base**, flow "Tìm kiếm/gợi ý tài liệu" ở trạng thái hiện tại — index tìm kiếm không áp cùng quy tắc cách ly như flow xem tài liệu, nên kết quả tìm kiếm (tiêu đề, số lượng kết quả) trả về xuyên phòng ban dù nội dung chi tiết vẫn bị chặn nếu nhân viên click vào xem. Flow này liên quan mật thiết tới enhance vì đây chính là lỗ hổng rò rỉ mà yêu cầu 4 của đề bài nhắm tới.

```mermaid
sequenceDiagram
    actor Emp as Nhân viên (phòng kinh doanh)
    participant App as Search Service
    participant Index as Search Index (toàn bộ DOCUMENT, không lọc phòng ban)

    Emp->>App: Gõ từ khóa "tái cơ cấu"
    App->>Index: Query full-text trên toàn bộ tài liệu công ty
    Index-->>App: Kết quả gồm cả tài liệu "Kế hoạch tái cơ cấu phòng kỹ thuật" (không thuộc phòng kinh doanh)
    App-->>Emp: Hiển thị tiêu đề + số lượng kết quả, bao gồm cả tài liệu không được phép xem nội dung
    Note over Emp,App: Nhân viên phòng kinh doanh biết được sự tồn tại của kế hoạch tái cơ cấu dù không mở được nội dung, đây đã là rò rỉ thông tin nhạy cảm
```
