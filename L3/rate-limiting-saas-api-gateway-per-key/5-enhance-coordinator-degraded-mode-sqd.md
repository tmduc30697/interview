# Enhance sequence — Redis/coordinator chậm hoặc down, fail-open có kiểm soát

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không phụ thuộc coordinator tập trung nên không có kịch bản nó down). Đáp ứng yêu cầu 4 của đề bài: khi Redis/coordinator lưu counter bị chậm/down tạm thời, phải có chiến lược rõ ràng thay vì hành vi không xác định.

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant GW as API Gateway
    participant Redis as Redis (RATE_LIMIT_BUCKET)
    participant Health as COORDINATOR_HEALTH_STATE
    participant Backend as Backend Service

    Customer->>GW: GET /api/resource
    GW->>Redis: Lấy token từ bucket
    Redis--xGW: Timeout / connection error

    GW->>Health: Cập nhật status=degraded, tăng failure streak

    alt Chọn chiến lược fail-open (mặc định cho SaaS này)
        Note over GW,Health: Ưu tiên availability hơn chặn nhầm khách hàng hợp lệ trong lúc coordinator down ngắn hạn, vì đa số request vẫn nằm trong hạn mức bình thường
        GW->>Backend: Forward request tạm thời, bỏ qua kiểm tra rate limit
        Backend-->>GW: Response
        GW-->>Customer: 200 OK, header X-RateLimit-Remaining=unknown
        GW->>GW: Ghi lại request này vào hàng đợi để đối soát lại usage khi Redis phục hồi
    else Chọn chiến lược fail-closed (khi cần bảo vệ nghiêm ngặt hơn, ví dụ nghi ngờ đang bị tấn công)
        GW-->>Customer: 503 Service Unavailable, không forward tới backend
        Note over GW: Chấp nhận chặn nhầm 1 số khách hàng hợp lệ trong thời gian ngắn để tránh backend bị tràn khi mất kiểm soát rate limit hoàn toàn
    end

    loop Định kỳ kiểm tra lại
        GW->>Redis: Thử kết nối lại
        Redis-->>GW: Phục hồi
        GW->>Health: Cập nhật status=healthy
    end
```
