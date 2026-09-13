# Enhance sequence — Gửi tin nhắn với token bucket cho phép burst, giữ nguyên nội dung khi bị chặn tạm

Đây là **enhance**, flow "Gửi tin nhắn" sau khi áp token bucket theo scope. So với base (ngưỡng cứng, mất nội dung khi chặn), flow này thay đổi ở 3 điểm: (1) dùng token bucket cho phép burst ngắn (vài tin liên tiếp trong 1-2 giây) miễn còn token, chỉ bắt pattern đều đặn kéo dài khi bucket cạn và refill_rate không theo kịp, (2) scope tách theo conversation (`per_conversation:<id>`) để gửi nhanh trong 1 group đông không tính chung với gửi rải rác tới nhiều người khác, (3) khi bị chặn tạm, chỉ ẩn nút gửi + đếm ngược vài giây, nội dung đang gõ vẫn được giữ nguyên trong ô nhập liệu.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat Service
    participant DB as RATE_LIMIT_BUCKET + RATE_LIMIT_POLICY store

    User->>App: Gửi tin nhắn 1 trong group đông đang thảo luận sôi nổi
    App->>DB: Đọc RATE_LIMIT_BUCKET(user, scope=per_conversation:group-42), tokens_remaining=3/3
    App->>DB: Trừ 1 token, tokens_remaining=2
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 2 (cách 0.5 giây)
    App->>DB: tokens_remaining=2, đủ token, trừ còn 1
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 3 (cách 0.3 giây, vẫn trong burst ngắn tự nhiên)
    App->>DB: tokens_remaining=1, đủ token, trừ còn 0
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 4 ngay lập tức (bucket đã cạn, refill_rate chưa kịp nạp lại)
    App->>DB: tokens_remaining=0, không đủ token

    App-->>User: Tạm ẩn nút gửi, hiện đếm ngược "chờ 1 giây", giữ nguyên nội dung tin nhắn 4 trong ô nhập liệu

    Note over DB: Sau đúng thời gian refill_rate quy định, bucket có lại token, user chỉ cần bấm gửi lại đúng nội dung đã gõ sẵn, không phải gõ lại từ đầu

    Note over App,DB: Nếu cùng nhịp gửi đều đặn này lặp lại liên tục hàng giờ (không có khoảng nghỉ tự nhiên của người thật), pattern đều đặn máy móc sẽ bị phát hiện ở flow 5-enhance-content-pattern-detection-sqd.md dù từng tin không vượt token bucket
```
