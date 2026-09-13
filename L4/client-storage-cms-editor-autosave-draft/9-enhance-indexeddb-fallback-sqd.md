# Enhance sequence — Fallback khi IndexedDB lỗi/không khả dụng

Đây là **enhance**, flow hoàn toàn mới, xử lý trường hợp IndexedDB bị chặn (ví dụ chế độ ẩn danh của một số trình duyệt) hoặc storage bị corrupt. Hệ thống phải phát hiện tình trạng này, chuyển sang fallback autosave thẳng lên server với tần suất thấp hơn, và cảnh báo rõ cho người dùng biết chế độ "an toàn khi crash cục bộ" đang không hoạt động. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant IDB as IndexedDB
    participant Health as STORAGE_HEALTH
    participant API as Article API

    Editor->>IDB: Thử mở/ghi thử một bản ghi kiểm tra khi khởi tạo editor
    alt IndexedDB khả dụng bình thường
        IDB-->>Editor: Ghi/đọc thành công
        Editor->>Health: Ghi indexeddb_available=true, fallback_mode_active=false
        Note over Editor: Tiếp tục dùng debounce autosave vào IndexedDB như flow edit-article
    else IndexedDB bị chặn hoặc lỗi/corrupt
        IDB-->>Editor: Lỗi khi mở database hoặc ghi thử thất bại
        Editor->>Health: Ghi indexeddb_available=false, fallback_mode_active=true
        Editor-->>User: Hiển thị cảnh báo rõ ràng, chế độ an toàn khi crash cục bộ đang không hoạt động
        loop Autosave fallback với tần suất thấp hơn nhiều so với debounce bình thường
            Editor->>API: Gửi thẳng content hiện tại lên server, chấp nhận tần suất thưa hơn
            API-->>Editor: Xác nhận đã lưu tạm trên server
        end
    end
```
