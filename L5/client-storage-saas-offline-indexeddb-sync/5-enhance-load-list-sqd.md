# Enhance sequence — Load list (stale-while-revalidate, có chỉ báo trạng thái sync)

Đây là **enhance**, cùng flow "load-list" như ở base nhưng thay đổi chiến lược: đọc từ IndexedDB hiển thị ngay lập tức, đồng thời gọi API ngầm để lấy bản mới nhất rồi cập nhật lại UI + IndexedDB. Vì dữ liệu hiển thị ban đầu có thể là stale, hệ thống phải có chỉ báo rõ ràng ("đang cập nhật..." hoặc thời điểm sync gần nhất) để user không nhầm tưởng đây là dữ liệu mới nhất. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (CACHED_RECORD)
    participant API as List API

    User->>App: Mở trang danh sách task/ticket/contact
    App->>IDB: Đọc CACHED_RECORD hiện có cho project này

    alt Có cache cục bộ
        IDB-->>App: Trả về danh sách đã cache, kèm last_synced_at
        App-->>User: Render ngay danh sách từ cache, hiển thị chỉ báo "đang cập nhật... (đồng bộ lần cuối lúc X)"
        Note over User: Nếu user bấm sửa ngay lúc này, UI phải nhắc rõ dữ liệu có thể chưa phải mới nhất
    else Không có cache (lần đầu mở)
        App-->>User: Hiển thị trạng thái đang tải
    end

    App->>API: Gọi API ngầm lấy dữ liệu mới nhất cho project
    API-->>App: Trả về danh sách mới nhất

    App->>IDB: Cập nhật CACHED_RECORD theo dữ liệu mới, is_stale=false, last_synced_at=now
    App-->>User: Cập nhật lại UI với dữ liệu mới nhất, gỡ chỉ báo "đang cập nhật...", hiển thị thời điểm sync mới
```
