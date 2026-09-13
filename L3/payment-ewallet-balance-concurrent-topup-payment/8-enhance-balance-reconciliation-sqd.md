# Enhance sequence — Job định kỳ đối chiếu số dư ví với lịch sử giao dịch

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — job định kỳ so sánh `WALLET.balance` với tổng cộng dồn từ `WALLET_LEDGER_ENTRY` (nạp, trừ, hoàn) của từng khách, nhằm bắt các trường hợp lệch hiếm gặp lọt qua cơ chế atomic (vd lỗi hạ tầng, bug nghiệp vụ khác), cảnh báo và tạm khóa giao dịch của tài khoản bị lệch trước khi khách phát hiện và khiếu nại.

```mermaid
sequenceDiagram
    participant Scheduler as Reconciliation Job (định kỳ)
    participant DB as WALLET + WALLET_LEDGER_ENTRY store
    participant Alert as BALANCE_RECONCILIATION_ALERT store
    participant Ops as Đội vận hành/điều tra

    loop Với mỗi WALLET
        Scheduler->>DB: Đọc WALLET.balance hiện tại
        Scheduler->>DB: Tính tổng cộng dồn delta_amount từ WALLET_LEDGER_ENTRY của wallet này
        Scheduler->>Scheduler: So sánh expected_balance (tổng ledger) với actual_balance (WALLET.balance)

        alt Khớp
            Scheduler->>Scheduler: Không làm gì, chuyển sang wallet tiếp theo
        else Lệch
            Scheduler->>Alert: Ghi BALANCE_RECONCILIATION_ALERT(expected, actual, status=open)
            Scheduler->>DB: UPDATE WALLET SET status = locked_for_investigation
            Alert-->>Ops: Gửi cảnh báo lệch số dư kèm chi tiết
            Ops->>Ops: Điều tra nguyên nhân lệch trước khi khách phát hiện và khiếu nại
        end
    end

    Note over Scheduler,DB: Job chỉ đọc và so sánh, không tự sửa balance, tránh che giấu lỗi race hiếm gặp mà chưa rõ nguyên nhân
```
