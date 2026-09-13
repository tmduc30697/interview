# Base sequence — Deploy (rolling trực tiếp, chưa có canary)

Đây là **base**, flow "Deploy checkout service" ở trạng thái hiện tại — đẩy thẳng 100% traffic sang bản mới, chỉ giám sát lỗi 5xx chung, rollback thủ công. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm cải tổ lại đúng flow deploy này cho riêng checkout.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Old as Deployment cũ
    participant Router as Router
    participant Dashboard as Monitoring Dashboard (chỉ 5xx chung)
    actor OnCall as On-call Engineer

    CI->>Old: Rolling deploy bản mới, thay thế dần instance cũ
    CI->>Router: Router trỏ 100% traffic sang bản mới ngay khi rolling xong
    Note over Router: Không có giai đoạn traffic nhỏ nào trước khi full 100% — mọi request checkout đều đi qua bản mới ngay lập tức
    Router-->>Dashboard: Traffic chảy qua bản mới
    Dashboard->>Dashboard: Chỉ theo dõi tỷ lệ lỗi 5xx tổng, không tách riêng theo version, không có metric thanh toán riêng
    Note over Dashboard,OnCall: Nếu payment gateway từ chối giao dịch nhưng service vẫn trả 200 OK, dashboard không phát hiện được gì bất thường
    OnCall->>Dashboard: Định kỳ kiểm tra thủ công
    OnCall->>CI: Nếu phát hiện vấn đề, tự tay kích hoạt rollback (mất vài phút)
```
