# Sequence Diagram — Enhance: Discover Instance

Đây là **enhance**, flow discover thay đổi so với base: caller nay dùng `CLIENT_CACHE_ENTRY` với TTL để giảm tải registry thay vì gọi trực tiếp mỗi lần, đồng thời trong lúc rolling deploy hai version cùng đăng ký gần như đồng thời thì caller vẫn phải nhận danh sách nhất quán, không bị lẫn instance cũ/mới gây lỗi.

```mermaid
sequenceDiagram
    actor Caller as Calling Service
    participant Registry as Service Registry
    participant Instance as Target Instance

    Caller->>Caller: Kiểm tra CLIENT_CACHE_ENTRY, còn hạn TTL
    alt cache còn hạn
        Caller->>Instance: Gọi instance từ cache, không hỏi registry
    else cache hết hạn hoặc chưa có
        Caller->>Registry: Get instances of Service X
        Registry->>Registry: Chốt snapshot nhất quán instance đang healthy tại thời điểm query
        Note over Registry: Trong lúc rolling deploy, trả về đúng 1 snapshot rõ ràng
        Note over Registry: Không lẫn giữa instance cũ đang bị loại và instance mới chưa healthy
        Registry-->>Caller: [Instance list] kèm ttl_expires_at
        Caller->>Caller: Lưu vào CLIENT_CACHE_ENTRY với TTL
        Caller->>Instance: Gọi instance
    end
    Instance-->>Caller: Response
```
