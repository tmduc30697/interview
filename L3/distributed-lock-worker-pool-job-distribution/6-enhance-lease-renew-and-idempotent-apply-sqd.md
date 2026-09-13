# Sequence Diagram - Enhance: lease-renew-and-idempotent-apply

Đây là flow **enhance mới**: job cần xử lý lâu hơn TTL ước tính (file lớn bất thường), worker phải renew lease định kỳ; lần renew giữa chừng thất bại do mất kết nối coordinator, lease hết hạn, worker khác nhận lại job trong khi worker cũ vẫn đang xử lý thật. Khi cả 2 cùng cố ghi kết quả, cơ chế kiểm tra idempotent (dựa trên `result_applied`/fencing_token) đảm bảo kết quả chỉ được áp dụng đúng 1 lần. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant W1 as Worker 1 (đang xử lý job lâu)
    participant Q as Job Queue/DB
    participant W2 as Worker 2

    W1->>Q: Claim job J1, lease_ttl=60s, fencing_token=201
    Q-->>W1: OK, expires_at=+60s
    loop Xử lý job kéo dài, renew định kỳ
        W1->>Q: Renew lease (gia hạn expires_at)
        Q-->>W1: OK
    end
    Note over W1,Q: Một lần renew bị mất kết nối tới coordinator, không tới nơi
    Q->>Q: Lease hết hạn vì không renew kịp
    Q->>Q: Trả job J1 về status=AVAILABLE
    W2->>Q: Poll job status=AVAILABLE
    Q-->>W2: Trả về job J1
    W2->>Q: Claim job J1, fencing_token=202
    Q-->>W2: OK, expires_at mới
    Note over W1: Worker 1 vẫn đang xử lý thật, không biết đã mất lease
    W1->>Q: Ghi kết quả job J1 kèm fencing_token=201
    Q->>Q: Kiểm tra fencing_token=201 đã cũ (không khớp token hiện hành 202)
    Q-->>W1: Từ chối ghi, báo lease đã mất, huỷ apply
    W2->>W2: Xử lý job J1 xong
    W2->>Q: Ghi kết quả job J1 kèm fencing_token=202, kiểm tra result_applied=false
    Q->>Q: result_applied=false, chấp nhận ghi, set result_applied=true
    Q-->>W2: OK, job J1 status=DONE
```
