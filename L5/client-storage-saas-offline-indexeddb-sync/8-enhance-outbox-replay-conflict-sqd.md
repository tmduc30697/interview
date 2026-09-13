# Enhance sequence — Replay outbox khi có mạng lại, xử lý conflict

Đây là **enhance**, flow hoàn toàn mới, tiếp nối flow "edit-record" khi mạng có lại. Các thao tác offline đã ghi trong outbox phải được replay đúng thứ tự (`order_seq`) lên server, và phải xử lý rõ trường hợp bản ghi đó đã bị người khác sửa trên server trong lúc offline. Đáp ứng phần sau của yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (OUTBOX_ENTRY)
    participant API as Record API
    participant DB as Server DB
    participant Conflict as SYNC_CONFLICT

    Note over App: Trình duyệt phát sự kiện online trở lại
    App->>IDB: Lấy toàn bộ OUTBOX_ENTRY status=pending, sắp theo order_seq tăng dần

    loop Với từng outbox entry theo đúng thứ tự
        App->>API: Gửi thao tác (update/delete) kèm version/updated_at mà local biết lúc bắt đầu sửa
        API->>DB: Kiểm tra RECORD.updated_at hiện tại trên server

        alt Không ai sửa bản ghi này trong lúc offline
            DB-->>API: Áp dụng thay đổi bình thường
            API-->>App: Xác nhận replay thành công
            App->>IDB: Đánh dấu OUTBOX_ENTRY.status=synced
        else Bản ghi đã bị người khác sửa trên server trong lúc offline
            DB-->>API: Phát hiện updated_at trên server mới hơn baseline local
            API-->>App: Trả về conflict, kèm bản ghi hiện tại trên server
            App->>Conflict: Ghi SYNC_CONFLICT(server_version_data, local_version_data)
            App->>IDB: Đánh dấu OUTBOX_ENTRY.status=conflict
            App-->>User: Hiển thị rõ xung đột, cho user chọn giữ bản của mình, giữ bản server, hay merge thủ công
        end
    end
```
