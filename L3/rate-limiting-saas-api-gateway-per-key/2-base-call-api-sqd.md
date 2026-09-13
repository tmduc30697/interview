# Base sequence — Gọi API (forward thẳng, không giới hạn)

Đây là **base**, flow "Gọi API" ở trạng thái hiện tại — gateway chỉ xác thực API key còn hiệu lực rồi forward thẳng tới backend, có log lại request nhưng không đối chiếu với bất kỳ giới hạn nào. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (một khách hàng dùng quá nhiều ảnh hưởng khách khác, không đúng với gói đã trả tiền) đều bắt nguồn từ việc thiếu bước kiểm tra giới hạn ở đây.

```mermaid
sequenceDiagram
    actor CustA as Khách hàng A (free plan)
    actor CustB as Khách hàng B (pro plan)
    participant GW as API Gateway (nhiều instance)
    participant Backend as Backend Service
    participant Log as API_REQUEST_LOG

    par Nhiều khách hàng gọi cùng lúc, không phân biệt gói
        CustA->>GW: GET /api/resource (kèm API key)
        GW->>GW: Xác thực API key còn active
        GW->>Backend: Forward request
        Backend-->>GW: Response
        GW->>Log: Ghi API_REQUEST_LOG
        GW-->>CustA: Response
    and
        CustB->>GW: Gọi API liên tục với tốc độ rất cao
        GW->>GW: Xác thực API key còn active
        GW->>Backend: Forward request (không giới hạn tốc độ)
        Backend-->>GW: Response
        GW->>Log: Ghi API_REQUEST_LOG
        GW-->>CustB: Response
    end

    Note over Backend: Khách hàng B dù chỉ trả tiền gói free vẫn có thể gọi vô hạn, chiếm hết tài nguyên backend làm chậm khách hàng A
```
