# Enhance sequence — Edit record (ghi vào outbox khi offline thay vì thất bại ngay)

Đây là **enhance**, cùng flow "edit-record" như ở base nhưng thay đổi hành vi khi mất mạng: thay vì thất bại ngay, thao tác sửa/xoá được ghi vào outbox trong IndexedDB kèm thứ tự thực hiện, để replay lên server khi có mạng lại (chi tiết replay và xử lý conflict ở flow outbox-replay-conflict riêng). Đáp ứng phần đầu của yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant API as Record API
    participant IDB as IndexedDB (CACHED_RECORD, OUTBOX_ENTRY)

    User->>App: Sửa nội dung 1 bản ghi (task/ticket/contact)
    App->>IDB: Cập nhật CACHED_RECORD cục bộ ngay để UI phản hồi tức thì

    alt Có mạng
        App->>API: PATCH record với nội dung mới
        API-->>App: Xác nhận đã lưu trên server
        App->>IDB: Cập nhật CACHED_RECORD.last_synced_at=now, is_stale=false
        App-->>User: Hiển thị đã lưu thành công
    else Mất mạng
        App->>IDB: Ghi OUTBOX_ENTRY(record_id, action=update, payload, order_seq tăng dần, status=pending)
        App-->>User: Hiển thị đã lưu tạm cục bộ, sẽ đồng bộ khi có mạng lại
        Note over IDB: Nội dung sửa không bị mất như ở base, đã nằm an toàn trong outbox chờ replay
    end
```
