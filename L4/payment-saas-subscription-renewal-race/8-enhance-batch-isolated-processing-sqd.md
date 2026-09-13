# Enhance sequence — Batch renewal xử lý độc lập, lỗi 1 subscription không lan sang subscription khác

Đây là **enhance**, flow mới phát sinh từ enhance — job renewal chạy hàng loạt cho nhiều subscription đến hạn trong ngày, mỗi subscription là 1 `RENEWAL_JOB_ITEM` độc lập với version lock riêng theo đúng flow ở file `5-enhance-renewal-billing-job-sqd.md`. Xử lý đúng yêu cầu 4 của đề bài: conflict hoặc lỗi ở 1 subscription không làm chậm hoặc ảnh hưởng tới việc xử lý các subscription khác trong cùng batch.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as RENEWAL_JOB_ITEM store
    participant GatewayA as Payment Gateway (sub A)
    participant GatewayC as Payment Gateway (sub C)

    Job->>DB: Tạo RENEWAL_JOB_RUN cho batch hôm nay, liệt kê các subscription đến hạn A, B, C

    par Xử lý subscription A độc lập
        Job->>DB: Lock A (version check), gọi GatewayA
        GatewayA-->>Job: Charge thành công
        Job->>DB: UPDATE RENEWAL_JOB_ITEM(A) status=success
    and Xử lý subscription B độc lập
        Job->>DB: Lock B (version check), phát hiện version đã đổi do khách vừa hủy
        Job->>DB: UPDATE RENEWAL_JOB_ITEM(B) status=skipped_cancelled
    and Xử lý subscription C độc lập
        Job->>DB: Lock C (version check), gọi GatewayC
        GatewayC-->>Job: Timeout/lỗi
        Job->>DB: UPDATE RENEWAL_JOB_ITEM(C) status=failed, ghi lại để retry riêng
    end

    Note over Job,DB: Subscription B bị skip và C bị lỗi không làm chậm hay chặn việc subscription A vẫn xử lý thành công đúng hạn, vì mỗi item có lock/version check và trạng thái riêng biệt
```
