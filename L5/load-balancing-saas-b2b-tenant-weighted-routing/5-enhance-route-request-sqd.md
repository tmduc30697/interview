# Enhance sequence — Route request (kiểm tra quota qua bộ đếm chia sẻ trước khi chọn backend)

Đây là **enhance** của flow `route-request` đã có ở base. So với base (chỉ round-robin, không biết tenant), giờ gateway phải: (1) đọc `TENANT_QUOTA_POLICY` để biết giới hạn và trọng số, (2) tăng/kiểm tra `RATE_LIMIT_COUNTER` trong store chia sẻ (không lưu cục bộ) để đếm đúng dù có nhiều gateway instance chạy song song, và (3) chọn backend có tính tới `route_weight` của tenant. Đáp ứng yêu cầu 1, yêu cầu 3, và một phần yêu cầu 4.

```mermaid
sequenceDiagram
    actor Tenant
    participant GW as Gateway Instance (bất kỳ instance nào trong cụm)
    participant SharedCounter as RATE_LIMIT_COUNTER store (Redis, chia sẻ toàn cụm)
    participant Policy as TENANT_QUOTA_POLICY
    participant BE as Backend Instance

    Tenant->>GW: Gửi request kèm API key
    GW->>Policy: Lấy quota hiện hành (max_rps, max_concurrent_connections, route_weight)
    Policy-->>GW: Trả policy của tenant
    GW->>SharedCounter: Atomic increment current_rps cho tenant (đếm chung toàn cụm, không riêng từng instance)
    SharedCounter-->>GW: current_rps sau khi tăng = X
    alt X vượt max_rps (đã tính cả burst_multiplier)
        GW-->>Tenant: 429 Too Many Requests, kèm thông tin quota hiện tại
    else Trong giới hạn quota
        GW->>GW: Chọn backend theo thuật toán chọn instance, có nhân thêm route_weight của tenant
        GW->>BE: Forward request
        BE-->>GW: Response
        GW-->>Tenant: Trả response
    end
    Note over GW,SharedCounter: Vì bộ đếm nằm ở store chia sẻ, dù request của cùng 1 tenant rơi vào các gateway instance khác nhau, tổng vẫn được tính đúng và không vượt ngưỡng thực tế
```
