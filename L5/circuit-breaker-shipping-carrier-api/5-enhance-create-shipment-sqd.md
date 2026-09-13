# Enhance sequence — Create shipment (breaker độc lập theo carrier + fallback)

Đây là **enhance**, cùng flow "create-shipment" như ở base nhưng thay đổi ở chỗ: mỗi carrier có breaker riêng nên breaker của carrier A mở không ảnh hưởng tới carrier B (yêu cầu 1), và khi breaker A mở hệ thống fail-fast rồi tự động chuyển sang carrier dự phòng theo thứ tự ưu tiên cấu hình sẵn (yêu cầu 2). So với base, request không còn gọi thẳng 1 carrier cố định mà luôn kiểm tra breaker riêng của carrier trước khi gọi.

```mermaid
sequenceDiagram
    actor OrderSvc as Order Service
    participant BreakerA as Breaker (Carrier A)
    participant CarrierA as Carrier A API
    participant Fallback as FALLBACK_PRIORITY config
    participant BreakerB as Breaker (Carrier B)
    participant CarrierB as Carrier B API
    participant DB as Shipment DB

    OrderSvc->>BreakerA: Kiểm tra state của breaker Carrier A
    alt Breaker A đang closed, bình thường
        OrderSvc->>CarrierA: Gọi API tạo vận đơn
        CarrierA-->>OrderSvc: Trả về tracking_code thành công
        OrderSvc->>DB: Cập nhật SHIPMENT status=created, carrier=A
    else Breaker A đang open, fail-fast ngay không gọi Carrier A
        Note over BreakerA: Breaker A độc lập với breaker của B/C, không khóa các carrier khác
        OrderSvc->>Fallback: Lấy carrier dự phòng theo priority (vùng, chi phí, SLA hiện tại)
        Fallback-->>OrderSvc: Carrier B là ưu tiên kế tiếp
        OrderSvc->>BreakerB: Kiểm tra state của breaker Carrier B
        alt Breaker B đang closed
            OrderSvc->>CarrierB: Gọi API tạo vận đơn ở carrier dự phòng
            CarrierB-->>OrderSvc: Trả về tracking_code thành công
            OrderSvc->>DB: Cập nhật SHIPMENT status=created, carrier=B, ghi nhận đây là đơn fallback
        else Breaker B cũng đang open
            OrderSvc->>Fallback: Thử tiếp carrier ưu tiên kế tiếp (carrier C)
        end
    end
```
