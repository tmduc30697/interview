# Sequence - Base: Tra cứu địa điểm gần đây (gọi Maps provider)

Đây là flow **base**: mobile app gọi backend để lấy danh sách địa điểm gần vị trí hiện tại, backend gọi ra dịch vụ bản đồ bên thứ ba. Toàn bộ thời gian (mạng di động, xử lý nội bộ, gọi Maps) bị gộp vào cùng một khối span cha, không tách biệt được phần nào là "chờ Maps" và phần nào là backend tự xử lý. Flow này là tiền đề cho yêu cầu 1 và 2 của đề bài (tách span external có tag provider, và tách latency mạng di động của client).

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant BE as Backend API
    participant Maps as Maps Provider (bên thứ ba)
    participant Tracer as Tracing Backend

    App->>BE: GET /nearby-places?lat&lng
    Note over BE: Trace được tạo, 1 span cha duy nhất bao trọn toàn bộ request
    BE->>BE: Validate request, đọc cache nội bộ
    BE->>Maps: Tìm địa điểm gần toạ độ (lat, lng)
    Maps-->>BE: Danh sách địa điểm
    Note over BE: Thời gian chờ Maps và thời gian xử lý nội bộ bị ghi chung vào 1 span, không tách được
    BE->>BE: Format kết quả trả về
    BE-->>App: 200 OK, danh sách địa điểm
    BE->>Tracer: Gửi span cha (duration tổng, không rõ phần nào là chờ Maps)
```
