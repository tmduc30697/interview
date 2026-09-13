# Enhance sequence — Request routing (tách pool theo loại traffic)

Đây là **enhance** của flow đã có ở base "Request routing". So với base, LB tra `POOL_ROUTING_WEIGHT` để đối xử khác nhau giữa request đọc (xem sản phẩm, có thể cache, ưu tiên rải đều/ít quan tâm nhất quán) và request ghi (mua hàng, cần nhất quán, ưu tiên instance ổn định), đáp ứng **yêu cầu 3** (không route request "mua hàng" và "xem sản phẩm" theo cùng một cách).

```mermaid
sequenceDiagram
    actor CustomerView as Customer xem sản phẩm
    actor CustomerBuy as Customer bấm mua hàng
    participant LB as Load Balancer
    participant Pool as POOL_ROUTING_WEIGHT
    participant InstanceA as Instance A (ổn định, ưu tiên cho buy)
    participant InstanceB as Instance B (chạy job nặng, chỉ nhận view)

    CustomerView->>LB: Request xem sản phẩm (type=view_product)
    LB->>Pool: Tra trọng số pool_type=view_product_read cho từng instance
    Pool-->>LB: InstanceB vẫn nhận được trọng số view hợp lý dù đang chạy job nặng, vì request đọc chấp nhận cache/độ trễ nhẹ
    LB-->>CustomerView: Route tới InstanceB

    CustomerBuy->>LB: Request mua hàng (type=checkout), gần như cùng lúc
    LB->>Pool: Tra trọng số pool_type=checkout_write cho từng instance
    Pool-->>LB: InstanceA có trọng số checkout_write cao hơn vì đang ổn định, cần nhất quán
    LB-->>CustomerBuy: Route tới InstanceA, tách biệt hẳn khỏi pool phục vụ view_product
    Note over LB,Pool: Cùng một cụm instance nhưng hai loại traffic được cân trọng số độc lập, tránh request ghi bị ảnh hưởng bởi tải của request đọc và ngược lại
```
