# Enhance sequence — Phát hiện bot qua nội dung lặp lại gửi tới nhiều đối tượng khác nhau

Đây là **enhance**, flow mới phát sinh từ enhance — phát hiện spam không chỉ dựa vào tần suất (vốn có thể luôn nằm dưới ngưỡng token bucket) mà còn dựa vào pattern nội dung: cùng 1 tin gửi liên tục tới nhiều group/user khác nhau trong thời gian ngắn với nhịp đều đặn máy móc. Đúng yêu cầu thứ 4 của đề bài — 1 tài khoản gửi đúng nhịp 1 tin/giây liên tục hàng giờ tới nhiều đối tượng khác nhau là dấu hiệu bot dù từng tin không vượt ngưỡng cứng.

```mermaid
sequenceDiagram
    actor Bot as Tài khoản bot
    participant App as Chat Service
    participant DB as MESSAGE + RATE_LIMIT_BUCKET store
    participant Detector as Content Spam Detector (background)
    participant Signal as CONTENT_SPAM_SIGNAL store
    participant Mod as Đội kiểm duyệt/hệ thống chặn tự động

    loop Mỗi giây, liên tục hàng giờ
        Bot->>App: Gửi cùng 1 nội dung tới 1 user/group khác nhau mỗi lần
        App->>DB: Kiểm tra RATE_LIMIT_BUCKET, luôn đủ token vì nhịp đúng 1 tin/giây không vượt burst
        App->>DB: Ghi MESSAGE(content_hash=giống nhau mỗi lần)
        App-->>Bot: Gửi thành công
    end

    par Detector chạy song song, quét theo content_hash
        Detector->>DB: Gom nhóm MESSAGE theo (sender_id, content_hash) trong cửa sổ thời gian gần đây
        Detector->>Detector: Tính distinct_target_count (số conversation/user khác nhau nhận cùng nội dung) và độ đều đặn của khoảng cách thời gian giữa các lần gửi
        Detector->>Detector: distinct_target_count cao + khoảng cách gần như cố định (đều đặn máy móc) => pattern_detected=regular_interval_bot
        Detector->>Signal: Ghi CONTENT_SPAM_SIGNAL(pattern_detected=regular_interval_bot)
    end

    Signal-->>Mod: Cảnh báo tài khoản nghi bot dù chưa từng vượt token bucket
    Mod->>App: Áp policy nghiêm ngặt hơn hoặc tạm khoá tài khoản để xác minh

    Note over Detector,Signal: Cơ chế này bắt được đúng trường hợp tần suất từng tin luôn hợp lệ nhưng tổng thể hành vi là bot rải tin hàng loạt
```
