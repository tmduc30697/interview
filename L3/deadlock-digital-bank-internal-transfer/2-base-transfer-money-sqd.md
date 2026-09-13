# Base sequence — Transfer money (lock theo thứ tự nguồn trước, đích sau, dẫn tới deadlock)

Đây là **base**, flow chuyển tiền ở trạng thái hiện tại: mỗi transaction lock account nguồn trước rồi mới lock account đích, theo đúng thứ tự tham số truyền vào chứ không chuẩn hóa. Đây chính là kịch bản gây deadlock kinh điển nêu ở yêu cầu 1 của đề bài, khi transaction A chuyển 1 → 2 và transaction B chuyển 2 → 1 chạy đồng thời.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    participant DB as Database
    participant Acc1 as ACCOUNT 1
    participant Acc2 as ACCOUNT 2

    UserA->>DB: BEGIN Transaction A (chuyển 1 → 2)
    UserB->>DB: BEGIN Transaction B (chuyển 2 → 1)

    DB->>Acc1: Transaction A: SELECT ... FOR UPDATE account 1 (nguồn)
    Acc1-->>DB: Lock granted cho Transaction A

    DB->>Acc2: Transaction B: SELECT ... FOR UPDATE account 2 (nguồn)
    Acc2-->>DB: Lock granted cho Transaction B

    DB->>Acc2: Transaction A: SELECT ... FOR UPDATE account 2 (đích) - chờ lock
    Note over Acc2: Account 2 đang bị Transaction B giữ lock

    DB->>Acc1: Transaction B: SELECT ... FOR UPDATE account 1 (đích) - chờ lock
    Note over Acc1: Account 1 đang bị Transaction A giữ lock

    Note over DB,Acc2: Transaction A chờ Transaction B, Transaction B chờ Transaction A, DEADLOCK
    DB-->>UserA: Lỗi deadlock, transaction bị rollback, không retry
    DB-->>UserB: Transaction còn lại tiếp tục và commit thành công
    DB-->>UserA: Trả lỗi "giao dịch thất bại" thẳng cho user, không log chi tiết
```
