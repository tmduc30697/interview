# Sequence Diagram - Enhance: realtime-latency-dashboard

Đây là flow **enhance mới**: một pipeline tổng hợp liên tục gom span từ toàn bộ order (không chỉ 1 order riêng lẻ) thành `STEP_LATENCY_ROLLUP` theo từng bước theo khung thời gian, để dashboard trả lời được câu hỏi "giờ cao điểm hôm nay chậm hơn hôm qua, chậm chủ yếu ở bước nào" bằng cách so sánh p50/p95/p99 theo thời gian thực. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant Spans as Luồng span từ mọi order (cart/inventory/payment/confirm)
    participant Aggregator as Rollup Aggregator
    participant Store as Step Latency Rollup Store
    participant Dash as Dashboard vận hành
    actor Ops as Đội vận hành

    loop Mỗi khung thời gian (ví dụ mỗi phút)
        Spans->>Aggregator: Đẩy toàn bộ span kết thúc trong khung thời gian này
        Aggregator->>Aggregator: Gom span theo step_name (cart/inventory/payment/confirm)
        Aggregator->>Aggregator: Tính p50, p95, p99 duration cho từng step
        Aggregator->>Store: Ghi STEP_LATENCY_ROLLUP theo step_name, time_bucket
    end
    Ops->>Dash: Mở dashboard, so sánh giờ cao điểm hôm nay với hôm qua
    Dash->>Store: Truy vấn rollup theo từng step, 2 khoảng thời gian
    Store-->>Dash: Trả về p50/p95/p99 từng step theo thời gian thực
    Dash-->>Ops: Hiển thị bước inventory-service có p95 tăng gấp đôi so với hôm qua, xác định đúng nút thắt
```
