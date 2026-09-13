# Sequence - Base: Mua vé theo số lượng (không chọn ghế, không giữ chỗ)

Đây là flow **base**: khách chọn số lượng vé muốn mua, điền thông tin thanh toán và thanh toán ngay trong 1 luồng liên tục, vé chỉ được trừ vào `remaining_tickets` sau khi thanh toán thành công. Vì không có ghế cụ thể, không cần bước "giữ chỗ tạm thời" nào cả — đây chính là điểm khác biệt nền tảng mà đề bài sẽ thay đổi khi thêm chọn ghế cụ thể.

```mermaid
sequenceDiagram
    participant U as Khách hàng
    participant API as Ticketing API
    participant Pay as Cổng thanh toán
    participant DB as Database

    U->>API: Chọn số lượng vé (vd 2 vé)
    U->>API: Điền thông tin thanh toán, xác nhận mua
    API->>Pay: Yêu cầu thanh toán
    Pay-->>API: Thanh toán thành công
    API->>DB: UPDATE remaining_tickets - 2 WHERE remaining_tickets >= 2
    DB-->>API: affected_rows=1
    API->>DB: Tạo ORDER(quantity=2, payment_status=paid)
    API-->>U: Mua vé thành công
    Note over DB: Không có bước giữ chỗ nào trước khi thanh toán, vì vé không gắn với 1 ghế cụ thể nào
```
