# Sequence Diagram — Base: Register Instance

Đây là **base**, flow một instance tự đăng ký khi khởi động — tiền đề bắt buộc để sau này bổ sung metadata version/zone, cơ chế deregister khi shutdown, và phát hiện crash qua TTL.

```mermaid
sequenceDiagram
    participant Instance as Service Instance
    participant Registry as Service Registry

    Instance->>Registry: Register (service_id, host, port)
    Registry->>Registry: Create INSTANCE record (status=healthy)
    Registry-->>Instance: Registration confirmed
```
