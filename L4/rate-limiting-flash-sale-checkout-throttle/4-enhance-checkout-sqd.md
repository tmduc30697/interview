# Enhance sequence — Checkout (qua phòng chờ, admission token miễn throttle lại)

Đây là **enhance** của flow "Checkout" đã có ở base. So với base, request không còn đi thẳng vào backend — user bị chặn ở virtual waiting room theo thứ tự FIFO (trừ khi có priority tier công bố trước), chỉ khi tới lượt mới nhận admission token; sau khi cầm token, user checkout thẳng qua backend mà không bị rate limit lần nữa (đáp ứng yêu cầu 1, 2 và 4 của đề bài).

```mermaid
sequenceDiagram
    actor VIP as User VIP
    actor Reg as User thường
    participant WR as Waiting Room Service
    participant Queue as WAITING_ROOM_QUEUE_ENTRY
    participant Token as ADMISSION_TOKEN
    participant Checkout as Checkout Service
    participant DB as Order/Inventory DB

    par Cao điểm, nhiều user cùng bấm mua
        Reg->>WR: POST /checkout
        WR->>Queue: Thêm QUEUE_ENTRY(priority_tier=regular, joined_at=now)
    and
        VIP->>WR: POST /checkout
        WR->>Queue: Thêm QUEUE_ENTRY(priority_tier=vip, joined_at=now)
    end

    Note over Queue: Thứ tự xử lý FIFO theo joined_at trong cùng priority_tier, VIP được công bố trước là ưu tiên hơn regular

    WR-->>Reg: Hiển thị vị trí hàng chờ và thời gian ước tính
    WR-->>VIP: Hiển thị vị trí hàng chờ (ngắn hơn nhờ priority)

    loop Tới lượt theo admit rate hiện tại
        Queue->>Token: Cấp ADMISSION_TOKEN cho entry đầu hàng
        Queue->>Queue: Đánh dấu QUEUE_ENTRY(status=admitted)
    end

    Token-->>VIP: Nhận token, chuyển vào luồng checkout
    VIP->>Checkout: POST /checkout (kèm admission_token)
    Checkout->>Token: Xác thực token còn hạn, chưa dùng
    Note over Checkout,Token: Token hợp lệ nên bỏ qua mọi rate limit khác, không bắt user chờ thêm lần nữa
    Checkout->>DB: Trừ flash_sale_stock, tạo ORDER
    DB-->>Checkout: Thành công
    Checkout-->>VIP: Checkout thành công
    Checkout->>Token: Đánh dấu used_for_checkout=true
```
