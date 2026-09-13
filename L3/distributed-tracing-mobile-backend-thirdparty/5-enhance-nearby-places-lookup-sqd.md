# Sequence - Enhance: Tra cứu địa điểm gần đây (tách client network + external span)

Đây là flow **enhance** của `nearby-places-lookup` (so với base). Khác biệt so với base: span cha được tách thành 1 đoạn network latency riêng của mobile app (client_send_ts vs server_receive_ts), và lời gọi Maps được bọc trong 1 span riêng gắn `span_kind=external`, `provider_name=maps`, tách hoàn toàn khỏi span xử lý nội bộ. Toạ độ chính xác của người dùng không được ghi thẳng vào tag mà chỉ ghi ở dạng đã làm tròn/ẩn danh theo `REDACTION_POLICY`. Đáp ứng **yêu cầu 1, 2 và 5** của đề bài.

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant BE as Backend API
    participant Maps as Maps Provider (bên thứ ba)
    participant Tracer as Tracing Backend

    App->>App: Gắn timestamp client_send_ts vào request
    App->>BE: GET /nearby-places?lat&lng
    Note over BE: server_receive_ts được ghi lại, CLIENT_NETWORK_SEGMENT = server_receive_ts - client_send_ts
    Note over BE: Span nội bộ (span_kind=internal) bắt đầu, tách riêng khỏi network segment
    BE->>BE: Validate request, đọc cache nội bộ
    Note over BE: Bắt đầu span external riêng, span_kind=external, provider_name=maps
    BE->>Maps: Tìm địa điểm gần toạ độ (đã làm tròn theo REDACTION_POLICY)
    Maps-->>BE: Danh sách địa điểm
    Note over BE: Kết thúc span external, duration chỉ tính riêng phần chờ Maps
    Note over BE: Toạ độ chính xác không được ghi vào tag, chỉ ghi status code và duration
    BE->>BE: Format kết quả trả về (vẫn trong span internal)
    BE-->>App: 200 OK, danh sách địa điểm
    BE->>Tracer: Gửi CLIENT_NETWORK_SEGMENT + span internal + span external riêng biệt
```
