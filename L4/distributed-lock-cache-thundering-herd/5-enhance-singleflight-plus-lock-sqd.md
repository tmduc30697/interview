# Enhance sequence — Singleflight nội bộ instance kết hợp distributed lock

Đây là **enhance**, flow hoàn toàn mới mô tả tầng singleflight bên trong 1 instance, đứng trước distributed lock ở tầng giữa các instance. Đáp ứng yêu cầu 3: nhiều request đồng thời trong cùng 1 instance cho cùng cache key chỉ tạo ra đúng 1 lời gọi tới coordinator (Redis), giảm số round-trip không cần thiết tới Redis khi vốn dĩ chúng đã có thể gộp chung kết quả ngay trong process.

```mermaid
sequenceDiagram
    actor Req1 as Request 1 (instance X)
    actor Req2 as Request 2 (instance X, cùng key)
    actor Req3 as Request 3 (instance X, cùng key)
    participant SF as Singleflight (in-process, instance X)
    participant Lock as Redis Lock
    participant DB as Database
    participant Cache as Cache Layer

    par 3 request cùng cache miss trong cùng 1 instance
        Req1->>SF: Cần dữ liệu article:123
        Req2->>SF: Cần dữ liệu article:123
        Req3->>SF: Cần dữ liệu article:123
    end

    SF->>SF: Phát hiện cùng key đang inflight, chỉ tạo 1 goroutine/task thực thi chung
    Note over SF: Req2, Req3 được gắn vào cùng 1 future/promise với Req1, không tạo request riêng

    SF->>Lock: (chỉ 1 lần) SET lock:article:123 NX PX 2000ms
    Lock-->>SF: Giành lock thành công

    SF->>DB: (chỉ 1 lần) SELECT * FROM ARTICLE WHERE id=123
    DB-->>SF: Kết quả

    SF->>Cache: SET article:123
    SF->>Lock: DEL lock:article:123

    SF-->>Req1: Trả kết quả
    SF-->>Req2: Trả kết quả (từ cùng 1 future, không gọi Redis/DB riêng)
    SF-->>Req3: Trả kết quả (từ cùng 1 future, không gọi Redis/DB riêng)

    Note over SF,Lock: Thay vì 3 lần round-trip tới Redis, singleflight gộp lại chỉ còn đúng 1 lần
```
