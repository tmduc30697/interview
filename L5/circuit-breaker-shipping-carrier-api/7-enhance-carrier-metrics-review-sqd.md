# Enhance sequence — Đo lường và đánh giá định kỳ theo từng carrier

Đây là **enhance**, flow hoàn toàn mới, đáp ứng yêu cầu 5 của đề bài: đo lường riêng theo từng carrier gồm tỉ lệ lỗi, thời gian breaker mở trong ngày, tỉ lệ đơn phải fallback sang carrier dự phòng, và chi phí phát sinh do fallback, dùng để đánh giá định kỳ có nên đổi thứ tự ưu tiên carrier hay không.

```mermaid
sequenceDiagram
    participant OrderSvc as Order Service
    participant BreakerA as Breaker (Carrier A)
    participant BreakerB as Breaker (Carrier B)
    participant Metric as CARRIER_METRIC store
    participant ReviewJob as Periodic Review Job
    participant Fallback as FALLBACK_PRIORITY config
    actor Ops as Vận hành / Logistics Analyst

    OrderSvc->>Metric: Mỗi lần gọi carrier, ghi kết quả thành công/lỗi để tính error_rate theo carrier
    OrderSvc->>Metric: Mỗi lần phải fallback sang carrier dự phòng, ghi nhận và tính thêm extra_cost (chênh lệch giá carrier dự phòng so với carrier chính)
    BreakerA-->>Metric: Mỗi lần chuyển trạng thái, cộng dồn breaker_open_duration_ms trong ngày
    BreakerB-->>Metric: Tương tự, đo riêng cho Carrier B

    loop Định kỳ cuối ngày
        ReviewJob->>Metric: Lấy error_rate, breaker_open_duration_ms, fallback_rate, extra_cost theo từng carrier
        Metric-->>ReviewJob: Trả về báo cáo tổng hợp theo từng carrier riêng biệt
        ReviewJob-->>Ops: Gửi báo cáo, ví dụ Carrier A lỗi nhiều giờ cao điểm, chi phí fallback sang Carrier B tăng
    end

    alt Ops quyết định đổi thứ tự ưu tiên carrier
        Ops->>Fallback: Cập nhật lại priority_order dựa trên SLA và chi phí thực tế gần đây
    else Giữ nguyên cấu hình hiện tại
        Ops->>Ops: Không thay đổi, tiếp tục theo dõi
    end
```
