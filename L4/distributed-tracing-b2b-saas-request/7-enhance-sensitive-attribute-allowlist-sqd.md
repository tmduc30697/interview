# Sequence Diagram - Enhance: sensitive-attribute-allowlist

Đây là flow **enhance mới**: khi `task-service` gắn attribute vào span (ví dụ thông tin request), tracing SDK kiểm tra từng key với `TRACE_ATTRIBUTE_ALLOWLIST` trước khi ghi - key hợp lệ như `task_id`, `project_id` được ghi bình thường, còn key nhạy cảm như `auth_token`/`password` bị chặn, không bao giờ xuất hiện trong tên span hay tag. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant TS as task-service
    participant SDK as Tracing SDK (local)
    participant Allow as Trace Attribute Allowlist
    participant Collector as Tracing Collector/Backend

    TS->>SDK: Gắn attribute vào span S2, key=task_id, value=T-123
    SDK->>Allow: Kiểm tra key=task_id có trong allowlist không
    Allow-->>SDK: Hợp lệ, cho phép ghi
    SDK->>Collector: Gửi span S2 kèm attribute task_id=T-123
    TS->>SDK: Gắn attribute vào span S2, key=auth_token, value=Bearer xyz...
    SDK->>Allow: Kiểm tra key=auth_token có trong allowlist không
    Allow-->>SDK: Không hợp lệ, key nhạy cảm bị chặn
    SDK->>SDK: Bỏ qua attribute auth_token, không đưa vào payload gửi đi
    Note over Collector: Collector không bao giờ nhận được giá trị auth_token/password trong span
```
