# Enhance sequence — Test giả lập 50 giao dịch ngược hướng, assert invariant tổng số dư

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 4 của đề bài: viết test giả lập 50 giao dịch chuyển tiền ngược hướng chạy song song giữa 2 account, rồi assert rằng tổng số dư 2 account trước và sau không đổi, dù có transaction phải retry hay không.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant Pool as 50 Transfer Transactions (25 chiều 1→2, 25 chiều 2→1)
    participant DB as Database (lock order + retry đã áp dụng)
    participant Acc1 as ACCOUNT 1
    participant Acc2 as ACCOUNT 2

    Test->>Acc1: Đọc balance ban đầu account 1
    Test->>Acc2: Đọc balance ban đầu account 2
    Test->>Test: Tính total_before = balance1 + balance2

    Test->>Pool: Khởi chạy đồng thời 50 transaction chuyển tiền ngược hướng
    par Chạy song song
        Pool->>DB: Transaction chiều 1→2 (có thể retry tối đa 3 lần)
    and
        Pool->>DB: Transaction chiều 2→1 (có thể retry tối đa 3 lần)
    end
    DB-->>Pool: Toàn bộ 50 transaction cuối cùng đều commit thành công (sau retry nếu cần)

    Test->>Acc1: Đọc balance sau cùng account 1
    Test->>Acc2: Đọc balance sau cùng account 2
    Test->>Test: Tính total_after = balance1 + balance2
    Test->>Test: assert total_after == total_before

    Note over Test,DB: Invariant phải đúng bất kể có bao nhiêu transaction phải retry do deadlock, vì retry chỉ làm chậm chứ không làm sai lệch số tiền chuyển
```
