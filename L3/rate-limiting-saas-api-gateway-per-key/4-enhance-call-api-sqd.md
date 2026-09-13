# Enhance sequence — Gọi API (token bucket dùng chung qua Redis, đúng chuẩn 429)

Đây là **enhance** của flow "Gọi API" đã có ở base. So với base, gateway không còn forward thẳng — mọi instance gateway cùng đọc/ghi chung 1 `RATE_LIMIT_BUCKET` trong Redis theo thuật toán token bucket (cho phép burst ngắn hợp lý) và kiểm tra thêm `MONTHLY_USAGE_COUNTER`, trả đúng chuẩn HTTP 429 kèm `Retry-After`/`X-RateLimit-Remaining` khi vượt (đáp ứng yêu cầu 1, 2 và 3 của đề bài).

```mermaid
sequenceDiagram
    actor CustA as Khách hàng A
    participant GW1 as API Gateway (instance 1)
    participant GW2 as API Gateway (instance 2)
    participant Redis as Redis (RATE_LIMIT_BUCKET dùng chung)
    participant Monthly as MONTHLY_USAGE_COUNTER
    participant Backend as Backend Service

    par 2 request gần như đồng thời, rơi vào 2 instance khác nhau
        CustA->>GW1: GET /api/resource
        GW1->>Redis: Lấy token từ bucket của API key (atomic decrement)
    and
        CustA->>GW2: GET /api/resource
        GW2->>Redis: Lấy token từ bucket của API key (atomic decrement)
    end

    Note over Redis: Counter tập trung ở Redis nên tổng số token bị trừ chính xác dù 2 instance xử lý song song, không đếm riêng lẻ từng instance

    alt Còn token trong bucket và chưa vượt requests_per_month
        Redis-->>GW1: Cho phép, trả tokens_remaining
        GW1->>Monthly: Tăng request_count trong tháng
        GW1->>Backend: Forward request
        Backend-->>GW1: Response
        GW1-->>CustA: 200 OK, header X-RateLimit-Remaining=tokens_remaining
    else Hết token trong bucket (vượt burst cho phép)
        Redis-->>GW2: Từ chối, tính thời gian tới lần refill token tiếp theo
        GW2-->>CustA: 429 Too Many Requests, header Retry-After, X-RateLimit-Remaining=0
    else Đã vượt requests_per_month dù còn token giây
        Monthly-->>GW1: Đã vượt hạn mức tháng
        GW1-->>CustA: 429 Too Many Requests, header Retry-After tới đầu tháng sau
    end
```
