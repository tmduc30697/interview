# Enhance sequence — Connection draining khi chuyển traffic

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (ở base, connection bị cắt ngang thẳng khi instance cũ bị dừng). Đáp ứng yêu cầu thứ 5 của đề bài: session/kết nối đang mở (WebSocket, request dài) tại thời điểm chuyển đổi không bị cắt ngang, mà hoàn tất trên Blue hoặc được chuyển tiếp có kiểm soát.

```mermaid
sequenceDiagram
    participant Router as Router/Load Balancer
    participant Blue as Blue Deployment
    participant Green as Green Deployment
    actor Client

    Note over Router: Ngay sau ROUTER_SWITCH_EVENT (traffic mới đã trỏ sang Green)
    Router->>Blue: Ngừng route request MỚI tới Blue, nhưng KHÔNG kill Blue ngay
    Blue->>Blue: Tạo CONNECTION_DRAIN (connection_type, count_at_switch = số CLIENT_CONNECTION đang mở)
    par Các connection đang mở tiếp tục chạy trên Blue
        Client->>Blue: WebSocket/long request đang mở tiếp tục nhận phản hồi bình thường
        Blue-->>Client: Hoàn tất connection tự nhiên trên Blue
    and Traffic mới hoàn toàn đi vào Green
        Client->>Green: Mọi request/connection mới được thiết lập với Green
    end
    Blue->>Blue: Theo dõi CONNECTION_DRAIN.drain_status cho tới khi count_at_switch về 0 hoặc hết drain timeout
    alt Toàn bộ connection hoàn tất trước timeout
        Blue->>Blue: drain_status=completed
    else Còn connection kéo dài quá timeout
        Blue->>Blue: drain_status=force_closed_after_timeout (chấp nhận cắt số ít còn lại, không chờ vô hạn)
    end
    Note over Blue: Chỉ sau khi drain_status=completed (hoặc force_closed), Blue mới đủ điều kiện chuyển sang status=retiring và bị tắt hẳn (xem flow "Deploy")
```
