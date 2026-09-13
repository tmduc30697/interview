# Enhance sequence — Route có trọng số theo gói dịch vụ khi cụm gần bão hòa

Đây là **enhance**, flow mới tập trung vào yêu cầu 4 của đề bài: khi cụm backend gần đạt tải tối đa, tenant trả phí cao hơn (`plan_tier` = premium) được ưu tiên nhiều tài nguyên hơn qua `route_weight`, mà không cần tách riêng instance backend vật lý cho từng nhóm tenant.

```mermaid
sequenceDiagram
    actor TenantPremium as Tenant Premium
    actor TenantFree as Tenant Free
    participant GW as Gateway Instance
    participant Policy as TENANT_QUOTA_POLICY
    participant LB as Weighted Instance Selector
    participant BE as Backend cluster (chung, gần bão hòa)

    Note over BE: Cụm backend đang ở gần capacity tối đa (vd 90% CPU)
    TenantPremium->>GW: Gửi request
    GW->>Policy: Lấy route_weight của Tenant Premium (vd weight=3)
    TenantFree->>GW: Gửi request cùng thời điểm
    GW->>Policy: Lấy route_weight của Tenant Free (vd weight=1)
    GW->>LB: Đưa cả 2 request vào hàng đợi chọn instance, có trọng số theo route_weight
    LB->>LB: Khi capacity hạn chế, ưu tiên cấp slot xử lý cho request có route_weight cao hơn trước
    LB->>BE: Forward request Tenant Premium trước
    BE-->>GW: Response nhanh cho Tenant Premium
    GW-->>TenantPremium: Trả response, độ trễ thấp
    LB->>BE: Forward request Tenant Free ngay sau khi có slot trống
    BE-->>GW: Response cho Tenant Free (độ trễ cao hơn 1 chút do phải chờ)
    GW-->>TenantFree: Trả response, vẫn thành công nhưng chậm hơn Premium
    Note over LB,BE: Cùng 1 cụm backend vật lý được chia sẻ, không có instance riêng cho từng plan_tier, chỉ khác nhau ở trọng số ưu tiên khi cụm gần bão hòa
```
