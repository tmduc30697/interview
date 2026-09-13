# Base sequence — Request routing

Đây là **base**, flow "Định tuyến request" ở trạng thái hiện tại: LB tự chuyển giữa round-robin (bình thường) và least-connection (khi có instance đang chạy job nặng), nhưng đối xử với mọi loại request (xem sản phẩm lẫn mua hàng) hoàn toàn như nhau, và đọc/ghi `active_connections` một cách đơn giản không đảm bảo atomic. Flow này là tiền đề cho **yêu cầu 3** (tách pool/trọng số theo loại traffic) và **yêu cầu 4** (race condition khi cập nhật bộ đếm tải) vì cả hai đều tác động trực tiếp lên đúng bước ra quyết định này.

```mermaid
sequenceDiagram
    actor CustomerView as Customer xem sản phẩm
    actor CustomerBuy as Customer bấm mua hàng
    participant LB as Load Balancer
    participant InstanceA as Instance A (đang chạy job nặng)
    participant InstanceB as Instance B (rảnh)

    CustomerView->>LB: Request xem sản phẩm
    LB->>LB: Kiểm tra current_algorithm, thấy có instance chạy job nặng nên dùng least_connection
    LB->>LB: Đọc active_connections của từng instance (đọc rồi cộng, không atomic)
    LB-->>CustomerView: Route tới InstanceB (ít connection hơn)

    CustomerBuy->>LB: Request mua hàng, gần như cùng lúc
    LB->>LB: Áp dụng đúng cùng thuật toán least_connection như request xem sản phẩm, không phân biệt loại traffic
    LB-->>CustomerBuy: Route tới InstanceB (cùng pool, cùng cách tính)
    Note over LB,InstanceB: Request đọc (xem sản phẩm) và request ghi (mua hàng, cần nhất quán) bị route lẫn lộn vào cùng instance theo cùng một cách, không có pool hay trọng số riêng
```
