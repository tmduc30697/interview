# Enhance sequence — Đổi gói/hủy gói biết trạng thái job renewal đang xử lý

Đây là **enhance**, flow "Khách đổi gói/hủy gói" sau khi phối hợp với version lock của job renewal. So với base (ghi đè trực tiếp không kiểm tra gì), flow này thay đổi ở chỗ: mọi thao tác đổi gói/hủy đều tăng `version` của subscription bất kể job có đang xử lý hay không (đây chính là tín hiệu để job phát hiện thay đổi ở file `5-enhance-renewal-billing-job-sqd.md`); riêng đổi gói (không phải hủy) khi job đang trong lúc `charging` sẽ ghi vào `pending_plan_id` thay vì đổi `plan_id` ngay, để tránh xung đột với giao dịch đang gọi cổng thanh toán.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Subscription Service
    participant DB as SUBSCRIPTION + RENEWAL_JOB_ITEM store

    Customer->>App: Bấm hủy subscription S
    App->>DB: Đọc RENEWAL_JOB_ITEM hiện tại của S (nếu có job đang chạy trong ngày)

    alt Job chưa lock hoặc chưa bắt đầu charge (status=pending, hoặc không có job chạy hôm nay)
        App->>DB: UPDATE SUBSCRIPTION SET status=cancelled, version=version+1
        App-->>Customer: "Đã hủy thành công"
    else Job đang locked/charging subscription này
        App->>DB: UPDATE SUBSCRIPTION SET status=cancelled, version=version+1 (vẫn ghi ngay, để job tự phát hiện version đổi và tự dừng ở bước kiểm tra trước khi charge)
        App-->>Customer: "Đã ghi nhận yêu cầu hủy"
    end

    Customer->>App: Bấm đổi gói Pro sang Basic
    App->>DB: Đọc RENEWAL_JOB_ITEM hiện tại của subscription

    alt Job không đang charging (chưa chạy, hoặc đã success/skipped)
        App->>DB: UPDATE SUBSCRIPTION SET plan_id=Basic, version=version+1
        App-->>Customer: "Đã đổi gói thành công, có hiệu lực ngay"
    else Job đang charging (đã gọi cổng thanh toán theo giá Pro, đang chờ phản hồi)
        App->>DB: UPDATE SUBSCRIPTION SET pending_plan_id=Basic, version=version+1 (không đổi plan_id ngay)
        App-->>Customer: "Đã ghi nhận đổi gói, có hiệu lực từ chu kỳ kế tiếp vì chu kỳ này đang được tính phí theo gói Pro"
    end
```
