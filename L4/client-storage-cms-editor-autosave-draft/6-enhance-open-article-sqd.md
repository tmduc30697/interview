# Enhance sequence — Open article (phát hiện draft cục bộ mới hơn bản server)

Đây là **enhance**, cùng flow "open-article" như ở base nhưng thêm bước kiểm tra draft trong IndexedDB trước khi hiển thị. Nếu phát hiện có draft cục bộ mới hơn bản đã lưu trên server (dấu hiệu của crash hoặc đóng nhầm tab trước đó), hệ thống phải hỏi rõ người dùng chọn khôi phục draft cục bộ hay giữ bản trên server, không tự động ghi đè ngầm định. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant IDB as IndexedDB (LOCAL_DRAFT)
    participant API as Article API

    User->>Editor: Mở lại bài viết (sau crash hoặc đóng nhầm tab)
    Editor->>API: GET article theo id
    API-->>Editor: Trả về content, updated_at (bản server)
    Editor->>IDB: Kiểm tra có LOCAL_DRAFT nào cho article này không

    alt Không có draft cục bộ, hoặc draft cục bộ cũ hơn/bằng bản server
        Editor-->>User: Hiển thị thẳng bản server, không cần hỏi gì thêm
    else Có draft cục bộ với updated_at_local mới hơn ARTICLE.updated_at trên server
        Editor-->>User: Hiển thị hộp thoại, phát hiện có nội dung chưa lưu mới hơn, khôi phục draft cục bộ hay giữ bản trên server
        alt User chọn khôi phục draft cục bộ
            Editor->>IDB: Đọc content_snapshot từ LOCAL_DRAFT
            Editor-->>User: Hiển thị nội dung từ draft cục bộ để soạn tiếp
        else User chọn giữ bản trên server
            Editor-->>User: Hiển thị nội dung từ server, bỏ qua draft cục bộ
            Note over Editor,IDB: Draft cục bộ cũ vẫn còn tồn tại tạm thời, sẽ được dọn ở flow cleanup
        end
    end
```
