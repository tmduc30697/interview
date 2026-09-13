# Enhance sequence — Backward-compatible rollback

Đây là **enhance**, cùng flow "Rollback" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 5 của đề bài: nếu canary đã ghi dữ liệu theo schema mới, rollback phải đảm bảo phiên bản cũ (đang phục vụ client legacy) vẫn đọc được dữ liệu đó, không chỉ đơn giản revert code.

```mermaid
sequenceDiagram
    participant Router as Router
    participant Canary as Backend canary
    participant Stable as Backend stable (bản cũ)
    participant Plan as SCHEMA_COMPATIBILITY_PLAN store
    participant DB as Shared Database

    Note over Router: ROLLBACK_EVENT vừa được kích hoạt (từ flow "Stratified canary monitoring")
    Router->>Plan: Tra SCHEMA_COMPATIBILITY_PLAN của backend_version canary
    Plan-->>Router: backward_read_strategy đã định nghĩa trước khi canary bắt đầu

    alt backward_read_strategy = ignorable_by_old_version
        Note over DB: Field mới canary đã ghi (vd trường mở rộng cho tính năng mới) được thiết kế để code cũ có thể bỏ qua an toàn, không gây lỗi parse
        Router->>Stable: Chuyển toàn bộ traffic về Stable
        Stable->>DB: Đọc dữ liệu bình thường, bỏ qua field mới không quen thuộc
        Stable-->>Router: Phục vụ đúng cho cả client cũ lẫn mới
    else backward_read_strategy = requires_shim
        Router->>Stable: Chuyển traffic về Stable, nhưng kèm 1 shim đọc dữ liệu
        Stable->>DB: Đọc dữ liệu (có thể ở schema mới)
        DB-->>Stable: Trả dữ liệu thô
        Stable->>Stable: Shim chuyển đổi dữ liệu về đúng hình dạng mà code cũ hiểu được
        Stable-->>Router: Phục vụ đúng, không gãy tương thích ngược lần nữa
    end
    Note over Plan: Vì SCHEMA_COMPATIBILITY_PLAN đã được chuẩn bị từ trước khi canary bắt đầu ghi dữ liệu mới, rollback không bị động lúc sự cố xảy ra
```
