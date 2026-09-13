# E-commerce đối soát giao dịch giữa cổng thanh toán và hệ thống đơn hàng

**Hệ thống:** Một sàn e-commerce nhận thanh toán qua cổng thanh toán bên thứ ba (ví dụ Stripe/VNPay) cho các đơn hàng.

**Vai trò của flow:** Đối soát định kỳ giữa danh sách giao dịch từ cổng thanh toán và trạng thái đơn hàng trong hệ thống nội bộ để phát hiện sai lệch (đơn báo đã thanh toán nhưng cổng không ghi nhận, hoặc ngược lại).

**Yêu cầu cụ thể:**
- Mỗi giao dịch từ cổng thanh toán phải được khớp (match) với đúng một đơn hàng nội bộ dựa trên mã tham chiếu duy nhất, phát hiện được các trường hợp không khớp (giao dịch không có đơn tương ứng, hoặc đơn báo đã trả tiền nhưng không tìm thấy giao dịch).
- Xử lý đúng các giao dịch bị hoàn tiền (refund) hoặc bị đảo (chargeback) xảy ra sau khi đơn hàng đã đối soát thành công trước đó — phải cập nhật lại trạng thái đối soát tương ứng, không để đơn hàng hiển thị "đã đối soát khớp" khi thực tế tiền đã được hoàn.
- Báo cáo đối soát phải phân loại rõ các loại sai lệch (thiếu giao dịch, thừa giao dịch, sai số tiền, sai trạng thái) để đội vận hành xử lý đúng quy trình cho từng loại, không gộp chung một danh sách "lỗi".
- Job đối soát phải chạy được lại nhiều lần trên cùng một khoảng thời gian dữ liệu mà không tạo ra báo cáo trùng lặp hoặc đánh dấu sai lệch nhiều lần cho cùng một giao dịch đã được xử lý.
- Các sai lệch không tự giải quyết được (cần con người can thiệp) phải được đưa vào hàng đợi xử lý thủ công có theo dõi trạng thái, đảm bảo không có sai lệch nào bị bỏ quên không xử lý.

---

# ERD — Base (trước khi có đối soát cổng thanh toán)

Đây là **base**: mô hình dữ liệu suy luận cho sàn e-commerce *trước khi* có flow đối soát. Đề bài giả định hệ thống đã có đơn hàng, đã tích hợp cổng thanh toán bên thứ ba và đã lưu lại các giao dịch cổng trả về (qua webhook) gắn với mã tham chiếu — nếu không có sẵn `ORDER` mang `payment_reference_code` và `PAYMENT_TRANSACTION` ghi nhận giao dịch cổng thì "khớp giao dịch cổng với đơn hàng" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ thanh toán đơn hàng, không suy diễn thêm các module không liên quan (khuyến mãi, vận chuyển...).

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o| PAYMENT_TRANSACTION : "paid via"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        decimal total_amount
        string currency
        string payment_reference_code
        string payment_status
        datetime created_at
    }

    PAYMENT_TRANSACTION {
        string transaction_id PK
        string gateway_reference_code
        string order_id FK
        decimal amount
        string currency
        string status
        datetime received_at
    }
```

---

# Sequence Diagram — Base: Checkout Payment

Đây là **base**, flow "thanh toán khi checkout" — tiền đề bắt buộc cho đối soát: chính flow này tạo ra `ORDER` với `payment_reference_code` và `PAYMENT_TRANSACTION` từ webhook của cổng thanh toán, hai dữ liệu mà job đối soát sau này phải khớp với nhau.

```mermaid
sequenceDiagram
    actor Customer
    participant Shop as E-commerce Service
    participant Gateway as Payment Gateway (Partner)

    Customer->>Shop: Checkout, confirm order
    Shop->>Shop: Create ORDER (status=pending_payment, generate payment_reference_code)
    Shop->>Gateway: Create charge request (payment_reference_code, amount)
    Gateway-->>Shop: Redirect customer to gateway payment page
    Customer->>Gateway: Enter payment info, confirm
    Gateway->>Gateway: Process payment

    Gateway-->>Shop: Webhook callback (gateway_reference_code, status, amount)
    Shop->>Shop: Create PAYMENT_TRANSACTION from webhook payload
    Shop->>Shop: Update ORDER.payment_status based on callback
    Shop-->>Customer: Show order confirmation

    Note over Shop,Gateway: Order's local payment_status and gateway's actual record can drift apart
    Note over Shop,Gateway: if webhook is lost, delayed, or delivered twice
```

---

# ERD — Enhance (sau khi có đối soát cổng thanh toán)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — chạy job đối soát khớp theo mã tham chiếu, phân loại rõ 4 loại sai lệch, xử lý refund/chargeback phát sinh sau khi đã khớp, đảm bảo job chạy lại (re-run) không tạo trùng lặp, và hàng đợi xử lý thủ công có theo dõi trạng thái cho sai lệch không tự giải quyết được. So với base, `ORDER` và `PAYMENT_TRANSACTION` không đổi cấu trúc, chỉ được tham chiếu thêm bởi các entity đối soát mới.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o| PAYMENT_TRANSACTION : "paid via"

    RECONCILIATION_RUN ||--o{ RECONCILIATION_RESULT : produces
    RECONCILIATION_RESULT |o--o| ORDER : "matches (optional)"
    RECONCILIATION_RESULT |o--o| PAYMENT_TRANSACTION : "matches (optional)"
    RECONCILIATION_RESULT ||--o| DISCREPANCY : "may yield"

    DISCREPANCY |o--o| MANUAL_REVIEW_ITEM : "escalated to"
    PAYMENT_TRANSACTION ||--o{ REFUND_EVENT : "may have"
    REFUND_EVENT |o--o| DISCREPANCY : "re-opens"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        decimal total_amount
        string currency
        string payment_reference_code
        string payment_status
        datetime created_at
    }

    PAYMENT_TRANSACTION {
        string transaction_id PK
        string gateway_reference_code
        string order_id FK
        decimal amount
        string currency
        string status
        datetime received_at
    }

    RECONCILIATION_RUN {
        string run_id PK
        date period_start
        date period_end
        datetime started_at
        datetime completed_at
        string status
    }

    RECONCILIATION_RESULT {
        string result_id PK
        string run_id FK
        string order_id FK
        string transaction_id FK
        string match_status
        datetime created_at
    }

    DISCREPANCY {
        string discrepancy_id PK
        string result_id FK
        string type
        decimal amount_diff
        string status
        datetime created_at
        datetime resolved_at
    }

    MANUAL_REVIEW_ITEM {
        string review_id PK
        string discrepancy_id FK
        string assigned_to
        string status
        datetime created_at
        datetime resolved_at
    }

    REFUND_EVENT {
        string refund_event_id PK
        string transaction_id FK
        string type
        decimal amount
        datetime occurred_at
        datetime processed_at
    }
```

---

<a id="4-enhance-periodic-reconciliation-sqd"></a>

# Sequence Diagram — Enhance: Periodic Reconciliation

Đây là **enhance**, flow chính hoàn toàn mới mà đề bài yêu cầu: đối soát định kỳ giữa `ORDER` và `PAYMENT_TRANSACTION`. Flow thể hiện việc khớp theo mã tham chiếu duy nhất, phân loại 4 loại sai lệch (thiếu giao dịch, thừa giao dịch, sai số tiền, sai trạng thái), và tính idempotent khi chạy lại nhiều lần trên cùng khoảng thời gian — tất cả chưa tồn tại ở base.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Shop as E-commerce Service
    participant Review as Manual Review Queue

    Ops->>Recon: Trigger reconciliation run (period_start, period_end)
    Recon->>Recon: Check for existing RECONCILIATION_RUN on this period

    alt run already exists and completed
        Recon-->>Ops: Reuse existing RECONCILIATION_RESULT, skip re-processing matched pairs
    else new or incomplete run
        Recon->>Recon: Create/resume RECONCILIATION_RUN

        loop for each ORDER with payment_status=paid in period
            Recon->>Shop: Look up PAYMENT_TRANSACTION by payment_reference_code
            alt matched, amount and status agree
                Shop-->>Recon: Transaction found, amount and status match
                Recon->>Recon: Create RECONCILIATION_RESULT (match_status=matched)
            else order paid but no transaction found
                Shop-->>Recon: No matching transaction
                Recon->>Recon: Create DISCREPANCY (type=missing_transaction)
            else transaction exists but order not marked paid
                Recon->>Recon: Create DISCREPANCY (type=missing_order_update)
            else amount differs
                Recon->>Recon: Create DISCREPANCY (type=amount_mismatch)
            else status differs
                Recon->>Recon: Create DISCREPANCY (type=status_mismatch)
            end
        end

        Recon->>Recon: Mark RECONCILIATION_RUN as completed
    end

    loop for each unresolved DISCREPANCY needing human judgment
        Recon->>Review: Create MANUAL_REVIEW_ITEM (status=open)
    end

    Recon-->>Ops: Reconciliation summary, discrepancies by type
```

---

<a id="5-enhance-handle-post-reconciliation-refund-sqd"></a>

# Sequence Diagram — Enhance: Handle Post-Reconciliation Refund/Chargeback

Đây là **enhance**, flow mới xử lý refund/chargeback xảy ra *sau khi* một đơn hàng đã đối soát khớp thành công (xem [4-enhance-periodic-reconciliation-sqd.md](#4-enhance-periodic-reconciliation-sqd)). Flow này đảm bảo yêu cầu "không để đơn hàng hiển thị đã đối soát khớp khi thực tế tiền đã được hoàn" — bằng cách mở lại DISCREPANCY cho RECONCILIATION_RESULT liên quan thay vì để trạng thái matched cũ đứng yên.

```mermaid
sequenceDiagram
    participant Gateway as Payment Gateway (Partner)
    participant Recon as Reconciliation Service
    participant Shop as E-commerce Service
    participant Review as Manual Review Queue

    Gateway-->>Recon: Webhook, refund or chargeback event (gateway_reference_code, amount)
    Recon->>Shop: Look up PAYMENT_TRANSACTION and matched ORDER
    Shop-->>Recon: Transaction found, order was previously reconciled as matched

    Recon->>Recon: Create REFUND_EVENT (linked to transaction_id)
    Recon->>Shop: Update ORDER.payment_status (refunded/charged_back)

    Recon->>Recon: Re-open DISCREPANCY on the related RECONCILIATION_RESULT (type=status_mismatch)
    Recon->>Review: Create MANUAL_REVIEW_ITEM if refund needs manual confirmation

    Recon-->>Gateway: Acknowledge webhook received
```

---

# Sequence Diagram — Enhance: Manual Review Queue

Đây là **enhance**, flow mới cho đội vận hành xử lý các `MANUAL_REVIEW_ITEM` phát sinh từ đối soát ([4-enhance-periodic-reconciliation-sqd.md](#4-enhance-periodic-reconciliation-sqd), [5-enhance-handle-post-reconciliation-refund-sqd.md](#5-enhance-handle-post-reconciliation-refund-sqd)). Flow này đảm bảo yêu cầu "không có sai lệch nào bị bỏ quên" bằng cách theo dõi trạng thái xử lý tường minh cho từng sai lệch cần con người can thiệp.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Review as Manual Review Queue
    participant Recon as Reconciliation Service
    participant Shop as E-commerce Service

    Ops->>Review: List open MANUAL_REVIEW_ITEM, ordered by discrepancy type
    Review-->>Ops: Item detail (discrepancy type, amount_diff, related order/transaction)

    Ops->>Review: Claim item, mark status=in_progress
    Ops->>Review: Investigate, decide resolution (adjust order, contact gateway support, ignore as noise)
    Ops->>Recon: Submit resolution

    Recon->>Shop: Apply correction if needed (update order status/amount)
    Recon->>Recon: Mark related DISCREPANCY as resolved
    Recon->>Review: Mark MANUAL_REVIEW_ITEM as resolved (resolved_at)

    Recon-->>Ops: Confirmation, item closed
```
