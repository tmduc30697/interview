# Base sequence — Rollback (thủ công, chưa có tiêu chí tự động)

Đây là **base**, flow "Xử lý khi bản deploy mới có lỗi" ở trạng thái hiện tại — hoàn toàn thủ công, phát hiện chậm. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 4 của đề bài chính là tự động hoá lại đúng flow này.

```mermaid
sequenceDiagram
    actor Users
    participant Dashboard as Monitoring Dashboard
    actor OnCall as On-call Engineer
    participant CI as CI/CD

    Users->>Users: Gặp lỗi 5xx sau khi bản mới lên
    Users-->>OnCall: Report qua ticket/Slack (không phải phát hiện tự động)
    OnCall->>Dashboard: Vào kiểm tra dashboard, xác nhận thủ công lỗi tăng bất thường
    OnCall->>OnCall: Điều tra, xác định bản deploy mới là nguyên nhân
    OnCall->>CI: Kích hoạt thủ công quy trình rolling deploy để quay lại bản cũ
    CI->>CI: Rolling deploy bản cũ (mất vài phút, giống flow Deploy)
    Note over Users,CI: Trong toàn bộ thời gian này (phát hiện + điều tra + rolling deploy lại), user vẫn gặp lỗi
```
