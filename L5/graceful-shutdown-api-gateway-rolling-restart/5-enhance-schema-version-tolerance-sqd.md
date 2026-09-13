# Enhance sequence — Chịu đựng schema khác nhau giữa version cũ và mới khi rolling restart dở dang

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 3** của đề bài: trong lúc rolling restart, một phần instance đã lên version mới (schema response có thêm/bớt field), phần còn lại vẫn version cũ — gateway và client không được crash vì sự khác biệt này.

```mermaid
sequenceDiagram
    actor Client
    participant GW as API Gateway
    participant Iold as Instance (version cũ, chưa restart)
    participant Inew as Instance (version mới, đã restart)

    Client->>GW: Request A
    GW->>Iold: Route request
    Iold-->>GW: 200 OK { order_id, status, total } (schema cũ, không có field discount_code)
    GW->>GW: Parse response bằng schema tolerant, field thiếu discount_code lấy giá trị mặc định null, không lỗi
    GW-->>Client: 200 OK { order_id, status, total, discount_code: null }

    Client->>GW: Request B
    GW->>Inew: Route request
    Inew-->>GW: 200 OK { order_id, status, total, discount_code, loyalty_points } (schema mới, thêm 2 field)
    GW->>GW: Parse response bằng schema tolerant, field lạ loyalty_points được giữ nguyên hoặc bỏ qua tuỳ hợp đồng API, không crash vì field thừa
    GW-->>Client: 200 OK { order_id, status, total, discount_code, loyalty_points }

    Note over GW,Inew: Gateway dùng chế độ parse "unknown fields allowed, missing optional fields = default" trong suốt thời gian INSTANCE.status của cả 2 version cùng tồn tại trong pool
```
