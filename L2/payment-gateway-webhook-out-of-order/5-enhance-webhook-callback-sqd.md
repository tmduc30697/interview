# Enhance sequence — Webhook idempotent + xác định thứ tự theo sequence number thật

Đây là **enhance**, flow "Nhận webhook từ cổng thanh toán" sau khi áp idempotency + ordering theo nguồn sự thật. So với base, flow này thay đổi ở 2 điểm: (1) kiểm tra `gateway_event_id` đã tồn tại trong `WEBHOOK_EVENT` chưa trước khi áp dụng thay đổi, webhook trùng chỉ trả 200 OK mà không làm gì thêm, (2) chỉ áp dụng thay đổi nếu `gateway_sequence_number` của webhook mới lớn hơn `last_applied_sequence_number` hiện tại — quyết định theo thứ tự thật phía cổng thanh toán, không theo thời điểm app nhận.

```mermaid
sequenceDiagram
    participant Gateway as Cổng thanh toán
    participant App as Order Service
    participant DB as WEBHOOK_EVENT + PAYMENT_TRANSACTION store

    Gateway->>App: Webhook payment_success (transaction_id=TX-1, gateway_event_id=EVT-1)
    App->>DB: Kiểm tra gateway_event_id=EVT-1 đã tồn tại chưa

    DB-->>App: Chưa tồn tại
    App->>DB: Ghi WEBHOOK_EVENT(EVT-1, applied=true), UPDATE PAYMENT_TRANSACTION status=success, last_applied_sequence_number=1
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-1, gateway_event_id=EVT-1) - retry do response chậm
    App->>DB: Kiểm tra gateway_event_id=EVT-1 đã tồn tại chưa
    DB-->>App: Đã tồn tại, applied=true
    App-->>Gateway: 200 OK (idempotent, không chạy lại side-effect)

    Note over Gateway,App: Trường hợp đảo thứ tự - webhook refunded (seq=2, xảy ra thật lúc T2) đến trước, webhook success cũ (seq=1, xảy ra thật lúc T1 < T2) đến sau do độ trễ mạng

    Gateway->>App: Webhook payment_refunded (transaction_id=TX-2, gateway_event_id=EVT-2, sequence=2)
    App->>DB: EVT-2 chưa tồn tại, sequence=2 > last_applied_sequence_number hiện tại (0)
    App->>DB: Ghi WEBHOOK_EVENT(EVT-2, applied=true), UPDATE PAYMENT_TRANSACTION status=refunded, last_applied_sequence_number=2
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-2, gateway_event_id=EVT-0, sequence=1, đến muộn)
    App->>DB: EVT-0 chưa tồn tại, nhưng sequence=1 <= last_applied_sequence_number hiện tại (2)
    App->>DB: Ghi WEBHOOK_EVENT(EVT-0, applied=false, lý do "đến muộn hơn event đã áp dụng"), không đổi PAYMENT_TRANSACTION
    App-->>Gateway: 200 OK

    Note over DB: Đơn hàng giữ đúng trạng thái refunded dù webhook success đến sau cùng, vì quyết định dựa trên sequence number thật chứ không theo thời điểm nhận
```
