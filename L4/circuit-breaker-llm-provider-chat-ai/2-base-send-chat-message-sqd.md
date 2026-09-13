# Base sequence — Send chat message (retry mù, không phân biệt lỗi)

Đây là **base**, flow "Gửi tin nhắn chat AI" ở trạng thái hiện tại — retry mọi loại lỗi như nhau, đoán thời gian chờ, không giới hạn rõ số lần. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat SaaS
    participant Provider as LLM Provider

    User->>App: Gửi tin nhắn
    App->>Provider: Gọi API sinh câu trả lời
    Provider-->>App: Lỗi 400 (prompt bị filter/input vượt giới hạn token)
    App->>Provider: Retry ngay (không phân biệt đây là lỗi do chính request của user)
    Provider-->>App: Vẫn lỗi 400 y hệt
    App->>Provider: Retry lần nữa, đoán delay cố định (không đọc header nào)
    Provider-->>App: Lỗi 429 (rate limit)
    App->>Provider: Tiếp tục retry theo delay đoán, có thể ngắn hơn provider yêu cầu
    Note over App,Provider: Mỗi lần retry đều tốn token/tiền dù lỗi 400 không bao giờ tự hết bằng cách gọi lại
    App-->>User: Sau nhiều lần thử, hiển thị lỗi chung chung hoặc user vẫn đang chờ loading
    Note over App: Không đo được đã tốn bao nhiêu chi phí/latency thêm do retry
```
