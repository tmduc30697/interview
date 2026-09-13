# Enhance sequence — Transfer money (lock theo account_id tăng dần, loại trừ deadlock)

Đây là **enhance** của flow `transfer-money` đã có ở base. So với base (lock theo thứ tự nguồn trước - đích sau gây deadlock), nay cả transaction A và B đều chuẩn hóa thứ tự lock theo `account_id` tăng dần trước khi lock, bất kể chiều chuyển tiền, đồng thời dùng isolation level `READ COMMITTED` tường minh. Đáp ứng yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    participant DB as Database (isolation=READ COMMITTED)
    participant Acc1 as ACCOUNT 1
    participant Acc2 as ACCOUNT 2

    UserA->>DB: BEGIN Transaction A (chuyển 1 → 2)
    UserA->>DB: Chuẩn hóa lock_order = sort([1,2]) = [1, 2]
    UserB->>DB: BEGIN Transaction B (chuyển 2 → 1)
    UserB->>DB: Chuẩn hóa lock_order = sort([2,1]) = [1, 2]

    DB->>Acc1: Transaction A: SELECT ... FOR UPDATE account 1 (theo lock_order)
    Acc1-->>DB: Lock granted cho Transaction A

    DB->>Acc1: Transaction B: SELECT ... FOR UPDATE account 1 (theo lock_order) - chờ lock
    Note over Acc1: Transaction B chờ Transaction A giải phóng account 1, không lock trước account 2

    DB->>Acc2: Transaction A: SELECT ... FOR UPDATE account 2 (theo lock_order)
    Acc2-->>DB: Lock granted cho Transaction A
    DB->>DB: Transaction A: trừ account 1, cộng account 2, COMMIT
    DB->>Acc1: Giải phóng lock account 1

    DB->>Acc1: Transaction B: nhận lock account 1 (vừa được giải phóng)
    Acc1-->>DB: Lock granted cho Transaction B
    DB->>Acc2: Transaction B: SELECT ... FOR UPDATE account 2 (theo lock_order)
    Acc2-->>DB: Lock granted cho Transaction B
    DB->>DB: Transaction B: trừ account 2, cộng account 1, COMMIT

    Note over DB,Acc2: Vì cả 2 transaction đều lock theo cùng thứ tự [1, 2], không bao giờ xảy ra vòng chờ chéo, deadlock được loại trừ hoàn toàn
```
