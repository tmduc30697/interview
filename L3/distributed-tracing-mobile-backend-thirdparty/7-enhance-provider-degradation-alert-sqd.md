# Sequence - Enhance: Phát hiện suy giảm chất lượng theo từng provider

Đây là flow **enhance hoàn toàn mới**, không tồn tại ở base (base chỉ có 1 cảnh báo chung "API chậm" gộp mọi nguyên nhân). Flow này mô tả hệ thống giám sát tổng hợp các span external theo `provider_name`, so với `PROVIDER_ALERT_RULE` riêng của từng provider (bản đồ, SMS, push notification), để phát hiện đúng provider nào đang suy giảm thay vì báo chung chung. Đáp ứng **yêu cầu 4** của đề bài.

```mermaid
sequenceDiagram
    participant Tracer as Tracing Backend
    participant Agg as Metrics Aggregator
    participant Rules as Provider Alert Rules
    participant Oncall as Đội vận hành

    loop Mỗi khoảng thời gian (vd 1 phút)
        Tracer->>Agg: Đẩy các span external (provider_name, duration, status)
        Agg->>Agg: Gom nhóm latency p95 và tỷ lệ lỗi theo từng provider_name riêng biệt
        Agg->>Rules: Lấy ngưỡng SLA của provider push (latency, error_rate)
        Rules-->>Agg: Ngưỡng riêng cho push khác với maps và sms
        alt Push notification vượt ngưỡng lỗi của riêng push
            Agg->>Oncall: Cảnh báo "Provider push đang suy giảm" kèm provider_name=push
            Note over Oncall: Biết ngay cần gây sức ép với đối tác push, không phải maps hay sms
        else Trong ngưỡng cho phép
            Agg->>Agg: Không phát cảnh báo
        end
    end
```
