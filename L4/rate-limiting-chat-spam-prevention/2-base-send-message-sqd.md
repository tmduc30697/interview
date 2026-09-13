# Base sequence — Gửi tin nhắn với rate limit ngưỡng cứng, chặn nhầm khi gõ dồn dập

Đây là **base**, flow "Gửi tin nhắn" ở trạng thái hiện tại — mỗi lần gửi, app cộng dồn bộ đếm theo cửa sổ thời gian cố định (vd 1 giây) áp dụng chung cho mọi user, không phân biệt gửi trong group đông hay chat 1-1, vượt ngưỡng là chặn ngay và mất luôn nội dung đang gõ. Flow này liên quan mật thiết tới enhance vì toàn bộ 4 yêu cầu của đề bài đều nhằm sửa đúng các hạn chế ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat Service
    participant DB as RATE_LIMIT_COUNTER + MESSAGE store

    User->>App: Gửi tin nhắn 1 trong group đang thảo luận sôi nổi
    App->>DB: Đọc RATE_LIMIT_COUNTER(user), message_count=1 trong cửa sổ hiện tại
    App->>DB: Ghi MESSAGE, tăng message_count=2
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 2 ngay sau đó (vẫn đang trò chuyện sôi nổi, cách 0.5 giây)
    App->>DB: message_count=3, vẫn dưới ngưỡng cứng 3 tin/giây
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 3 (cách 0.3 giây, vẫn là người thật gõ nhanh)
    App->>DB: message_count=4, vượt ngưỡng cứng 3 tin/giây
    App-->>User: Từ chối gửi, ẩn nút gửi, xoá luôn nội dung đang gõ trong ô nhập liệu

    Note over App,DB: Ngưỡng cố định không phân biệt burst tự nhiên của user thật đang sôi nổi trong group đông với pattern đều đặn của bot, và làm mất nội dung tin nhắn khi chặn
```
