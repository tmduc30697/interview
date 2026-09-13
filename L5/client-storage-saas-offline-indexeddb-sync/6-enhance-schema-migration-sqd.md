# Enhance sequence — Migration schema IndexedDB giữa các version app

Đây là **enhance**, flow hoàn toàn mới, xử lý việc đổi schema lưu trong IndexedDB giữa các version app (thêm field, đổi cấu trúc object store). Phải dùng cơ chế `onupgradeneeded` để migrate mà không làm mất dữ liệu cache cũ, và không được crash app cho user đang chạy version cũ chưa kịp cập nhật. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard (version mới)
    participant IDB as IndexedDB
    participant Meta as SCHEMA_VERSION

    User->>App: Mở app sau khi đã cập nhật lên version mới
    App->>IDB: Mở database, yêu cầu db_version mới (ví dụ tăng từ 3 lên 4)

    alt db_version trên máy user cũ hơn version app yêu cầu
        IDB->>App: Kích hoạt sự kiện onupgradeneeded
        App->>IDB: Chạy migration, ví dụ thêm field mới vào object store hoặc tạo index mới, giữ nguyên dữ liệu cũ
        App->>Meta: Cập nhật SCHEMA_VERSION(db_version=4, migration_notes, migrated_at=now)
        IDB-->>App: Mở database thành công với schema mới, dữ liệu CACHED_RECORD cũ vẫn còn nguyên
    else db_version đã khớp
        IDB-->>App: Mở database bình thường, không cần migrate
    end

    App-->>User: Tiếp tục dùng app bình thường với dữ liệu cache cũ đã được migrate

    Note over App: Trường hợp riêng, user khác vẫn đang chạy version app cũ (chưa update)
    App->>IDB: Version app cũ mở database với db_version cũ hơn bản đã được version mới nâng lên
    IDB-->>App: Trình duyệt từ chối mở ở version thấp hơn version hiện có của database
    App-->>User: App version cũ phát hiện không mở được cache, tự chuyển sang gọi thẳng API thay vì crash
```
