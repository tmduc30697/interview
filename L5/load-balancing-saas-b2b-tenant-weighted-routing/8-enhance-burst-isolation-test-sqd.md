# Enhance sequence — Test 1 tenant tăng traffic 50x, đo ảnh hưởng tới tenant khác

Đây là **enhance**, flow test mô tả ở yêu cầu 5 của đề bài: mô phỏng 1 tenant tăng traffic đột biến 50 lần trong vài giây, đo độ trễ và tỷ lệ lỗi của các tenant khác trong cùng cụm, chứng minh giới hạn tài nguyên theo tenant (yêu cầu 1, 3) đã cô lập được ảnh hưởng. Đối lập trực tiếp với flow base `tenant-overload-incident` — cùng kịch bản traffic nhưng kết quả khác hẳn nhờ enhance.

```mermaid
sequenceDiagram
    participant TestRunner as Test Runner
    actor TenantBig as Tenant lớn (traffic x50 trong vài giây)
    actor TenantSmall as Tenant nhỏ (traffic bình thường, baseline)
    participant GW as Gateway cluster (nhiều instance)
    participant SharedCounter as RATE_LIMIT_COUNTER store
    participant BE as Backend cluster

    TestRunner->>TenantSmall: Ghi nhận baseline latency/error_rate của Tenant nhỏ trước test
    TestRunner->>TenantBig: Bắt đầu bơm traffic tăng 50x trong vài giây
    TenantBig->>GW: Traffic tăng đột biến, rải qua nhiều gateway instance khác nhau
    GW->>SharedCounter: Mỗi request đều tăng bộ đếm chia sẻ của Tenant lớn (tổng đúng dù rải nhiều instance)
    SharedCounter-->>GW: Phần vượt max_rps/burst bị đánh dấu throttled/blocked
    GW-->>TenantBig: Phần lớn traffic vượt ngưỡng bị 429, chỉ phần trong quota được forward
    TenantSmall->>GW: Vẫn gửi traffic bình thường như baseline
    GW->>BE: Forward đầy đủ request của Tenant nhỏ (không bị ảnh hưởng vì Tenant lớn đã bị chặn ở tầng gateway)
    BE-->>GW: Response đúng thời gian như baseline
    GW-->>TenantSmall: Response thành công, latency gần như baseline
    TestRunner->>TestRunner: So sánh latency/error_rate của Tenant nhỏ trước và trong lúc Tenant lớn bị spike
    TestRunner-->>TestRunner: Assert chênh lệch không đáng kể (vd dưới 5%), chứng minh cô lập tài nguyên hoạt động đúng
```
