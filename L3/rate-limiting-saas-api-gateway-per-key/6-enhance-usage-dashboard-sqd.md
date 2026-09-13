# Enhance sequence — Dashboard khách hàng tự xem usage so với limit gói

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có nơi nào để khách hàng xem usage vì chưa có khái niệm limit). Đáp ứng yêu cầu 5 của đề bài: khách hàng tự xem usage hiện tại so với limit gói của họ, chủ động nâng cấp trước khi bị chặn.

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant Portal as Customer Portal
    participant API as Usage API
    participant Bucket as RATE_LIMIT_BUCKET
    participant Monthly as MONTHLY_USAGE_COUNTER
    participant Plan as PLAN_LIMIT

    Customer->>Portal: Mở trang "Usage & Limits"
    Portal->>API: GET /usage?api_key_id=...

    API->>Plan: Đọc requests_per_second, requests_per_month theo plan hiện tại
    API->>Bucket: Đọc tokens_remaining, bucket_capacity gần thời gian thực
    API->>Monthly: Đọc request_count tháng hiện tại

    API-->>Portal: Trả về usage hiện tại vs limit gói
    Portal-->>Customer: Hiển thị biểu đồ req/giây gần đây, và thanh tiến độ request_count/requests_per_month

    alt Usage tháng đang tiến gần requests_per_month, ví dụ trên 80%
        Portal-->>Customer: Gợi ý nâng cấp gói trước khi bị chặn 429 do vượt hạn mức tháng
    end
```
