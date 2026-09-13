# Enhance sequence — Đo throughput và tỉ lệ user phải chờ để lên kế hoạch capacity

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không đo lường throughput hay tỉ lệ user bị chặn). Đáp ứng yêu cầu 5 của đề bài: đo throughput checkout thành công/giây tối đa an toàn, và tỉ lệ user phải vào hàng chờ so với tổng traffic, phục vụ capacity planning cho các sự kiện sau.

```mermaid
sequenceDiagram
    participant Checkout as Checkout Service
    participant WR as Waiting Room Service
    participant Reporter as Metrics Reporter (job định kỳ)
    participant Metric as THROUGHPUT_METRIC
    actor Planner as Đội capacity planning

    loop Suốt giờ flash sale
        Checkout->>Reporter: Ghi nhận mỗi checkout thành công
        WR->>Reporter: Ghi nhận mỗi user vào hàng chờ / được admit
    end

    loop Cuối mỗi window (ví dụ mỗi phút)
        Reporter->>Reporter: Tính successful_checkouts_per_sec trong window
        Reporter->>Reporter: Tính queued_ratio_pct = queued_user_count / total_user_count
        Reporter->>Metric: Ghi THROUGHPUT_METRIC(window_start, window_end, successful_checkouts_per_sec, queued_ratio_pct)
    end

    Planner->>Metric: Truy vấn throughput tối đa an toàn đã đạt được trong sự kiện
    Metric-->>Planner: Trả về throughput đỉnh và tỉ lệ user từng phải chờ

    Note over Planner,Metric: Dữ liệu này dùng để ước tính backend cần scale thêm bao nhiêu, hoặc admit rate nên đặt bao nhiêu cho flash sale kế tiếp
```
