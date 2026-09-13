# Enhance sequence — Eviction khi gần chạm quota IndexedDB

Đây là **enhance**, flow hoàn toàn mới, xử lý giới hạn dung lượng IndexedDB theo quota per-origin của trình duyệt. Khi gần chạm quota, hệ thống phải áp dụng chiến lược eviction LRU theo project ít dùng nhất, và tránh để lỗi ghi thất bại giữa chừng làm hỏng transaction đang dở. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (CACHED_RECORD)
    participant Quota as STORAGE_QUOTA_STATE
    participant Evict as EVICTION_LOG

    App->>Quota: Kiểm tra dung lượng đã dùng so với quota per-origin trước khi ghi cache mới
    alt Còn nhiều dung lượng trống
        App->>IDB: Ghi CACHED_RECORD bình thường trong 1 transaction
        IDB-->>App: Ghi thành công
    else Gần chạm quota (ví dụ trên 90%)
        App->>Quota: Xác định các project có last_accessed_at cũ nhất (LRU)
        Quota-->>App: Trả về danh sách project ít dùng nhất cần giải phóng

        App->>IDB: Mở transaction riêng để xoá CACHED_RECORD của các project đó, tách biệt với transaction ghi dữ liệu mới
        IDB-->>App: Xoá thành công, giải phóng dung lượng
        App->>Evict: Ghi EVICTION_LOG(project_id, freed_bytes, reason=quota_near_full)

        App->>IDB: Sau khi đã giải phóng đủ, mới mở transaction mới để ghi CACHED_RECORD cần lưu
        alt Ghi thành công
            IDB-->>App: OK
        else Vẫn thất bại do quota (trường hợp hiếm)
            IDB-->>App: Lỗi ghi, transaction tự rollback nguyên vẹn nhờ tách transaction riêng từ bước xoá
            App-->>User: Bỏ qua cache cho lần này, vẫn hiển thị dữ liệu vừa gọi API, không để transaction dở dang làm hỏng cache hiện có
        end
    end
```
