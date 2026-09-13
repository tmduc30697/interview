# Enhance sequence — Fencing token chặn ghi trừ kho từ process đã mất lock

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp process bị GC pause/network delay tưởng mình còn giữ lock. Đáp ứng yêu cầu 3: dùng fencing token gắn với lock để tránh process bị treo ghi trừ kho sau khi lock đã hết hạn và bị process khác giành mất — thao tác trừ kho với token cũ phải bị inventory service từ chối.

```mermaid
sequenceDiagram
    participant ProcA as Process A (bị GC pause)
    participant ProcB as Process B
    participant Lock as Redis Lock (lock:sku:SKU-123)
    participant InvSvc as Inventory Service
    participant DB as Database

    ProcA->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+1
    Lock-->>ProcA: Giành lock thành công

    ProcA->>ProcA: Bắt đầu xử lý trừ kho, bị GC pause dài giữa chừng

    Note over Lock: TTL 300ms hết hạn trong lúc ProcA bị pause, chưa kịp renew hay release

    ProcB->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+2
    Lock-->>ProcB: Giành lock thành công vì lock cũ đã hết hạn

    ProcB->>InvSvc: Trừ kho SKU-123 kèm fencing_token=N+2
    InvSvc->>InvSvc: Kiểm tra N+2 >= last_accepted_token (N+1), hợp lệ
    InvSvc->>DB: UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    InvSvc->>DB: INSERT ORDER (fencing_token_used=N+2, status=confirmed)
    InvSvc->>InvSvc: Cập nhật last_accepted_token=N+2
    InvSvc-->>ProcB: Mua thành công

    Note over ProcA: ProcA tỉnh lại sau GC pause, vẫn tưởng còn giữ lock với fencing_token=N+1
    ProcA->>InvSvc: Trừ kho SKU-123 kèm fencing_token=N+1 (đã trễ)

    InvSvc->>InvSvc: Kiểm tra N+1 < last_accepted_token (N+2), token đã cũ
    InvSvc-->>ProcA: Từ chối, "fencing token expired, lock đã đổi chủ, không thực hiện trừ kho"

    Note over InvSvc,DB: Nhờ vậy tồn kho không bị trừ 2 lần dù cả 2 process đều tưởng mình đang giữ lock hợp lệ
```
