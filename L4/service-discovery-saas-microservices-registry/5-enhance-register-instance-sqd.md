# Sequence Diagram — Enhance: Register Instance

Đây là **enhance**, flow đăng ký thay đổi so với base: instance nay gửi kèm version/zone, tự deregister khi nhận shutdown signal, và nếu crash không kịp deregister thì registry tự phát hiện qua TTL/heartbeat timeout trong SLA xác định (ví dụ dưới 30 giây) thay vì instance chết luôn nằm trong registry.

```mermaid
sequenceDiagram
    participant Instance as Service Instance
    participant Registry as Service Registry

    Instance->>Registry: Register (service_id, host, port, version, zone)
    Registry->>Registry: Create INSTANCE (status=healthy, ttl_expires_at=now+TTL)
    Registry-->>Instance: Registration confirmed

    loop heartbeat mỗi chu kỳ
        Instance->>Registry: Passive heartbeat
        Registry->>Registry: Gia hạn ttl_expires_at
    end

    alt graceful shutdown
        Instance->>Registry: Deregister trước khi tắt
        Registry->>Registry: Remove INSTANCE record ngay lập tức
    else crash đột ngột (kill -9, mất điện)
        Instance--xRegistry: Không còn heartbeat, không deregister
        Registry->>Registry: Phát hiện ttl_expires_at đã qua
        Registry->>Registry: Loại instance khỏi danh sách active trong SLA phát hiện
    end
```
