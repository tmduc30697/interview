# Base sequence — Sự cố 1 tenant lớn làm nghẽn tenant khác (chưa có cô lập tài nguyên)

Đây là **base**, mô tả sự cố xảy ra khi chưa có giới hạn theo tenant: một tenant lớn tăng traffic đột biến, chiếm hết capacity chung của backend, khiến các tenant nhỏ khác cũng bị chậm/timeout dù họ không hề tăng traffic. Đây chính là vấn đề gốc mà toàn bộ đề bài (giới hạn, phân loại traffic, trọng số theo gói) phải giải quyết.

```mermaid
sequenceDiagram
    actor TenantBig as Tenant lớn (traffic tăng đột biến)
    actor TenantSmall as Tenant nhỏ (traffic bình thường)
    participant GW as Gateway Instance
    participant BE as Backend cluster (capacity chung)

    TenantBig->>GW: Gửi lượng request tăng đột biến (không giới hạn)
    GW->>BE: Forward toàn bộ request của Tenant lớn (không kiểm tra quota)
    BE->>BE: Capacity chung gần bão hòa vì request của Tenant lớn
    TenantSmall->>GW: Gửi request bình thường như mọi khi
    GW->>BE: Forward request của Tenant nhỏ
    BE-->>GW: Response chậm/timeout do backend quá tải chung
    GW-->>TenantSmall: Trả lỗi timeout dù Tenant nhỏ không hề tăng traffic
    Note over BE,TenantSmall: Vì không có giới hạn/cô lập theo tenant, 1 tenant lớn có thể chiếm hết capacity chung và ảnh hưởng toàn bộ tenant khác
```
