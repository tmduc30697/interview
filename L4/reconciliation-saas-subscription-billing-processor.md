# SaaS thuê bao đối soát billing nội bộ với payment processor

**Hệ thống:** Một SaaS thu phí thuê bao định kỳ hàng tháng, khách hàng có thể nâng/hạ cấp gói giữa chu kỳ, thanh toán xử lý qua một payment processor bên ngoài.

**Vai trò của flow:** Đối soát giữa hệ thống billing nội bộ (invoice phát sinh, các khoản điều chỉnh proration khi đổi gói) và số tiền thực tế processor đã thu được/báo cáo lại theo từng chu kỳ thanh toán.

**Yêu cầu cụ thể:**
- Khi khách hàng đổi gói giữa chu kỳ, hệ thống billing nội bộ phát sinh nhiều invoice điều chỉnh nhỏ (giảm trừ phần chưa dùng của gói cũ, cộng thêm phần gói mới) trong cùng một khoảng thời gian ngắn — đối soát phải khớp đúng tổng các điều chỉnh này với đúng một lượt thu tiền tương ứng từ processor, tránh hiểu nhầm là nhiều giao dịch riêng lẻ không khớp.
- Processor có thể thử thu tiền thất bại rồi tự động retry theo lịch riêng của họ (dunning) trong vài ngày sau ngày đến hạn — đối soát phải xử lý được trạng thái "đang chờ retry" khác với "thất bại hẳn", không đóng sổ chu kỳ khi khoản thu vẫn còn khả năng thành công ở lần retry tiếp theo.
- Khách hàng hủy gói giữa chu kỳ có thể được hoàn lại phần tiền chưa dùng, khoản hoàn này phát sinh sau khi invoice gốc đã đối soát khớp từ trước — cần cập nhật lại trạng thái đối soát của invoice đó, không để hệ thống báo cáo doanh thu vẫn tính khoản đã hoàn là doanh thu thực.
- Số tiền processor thực báo cáo có thể đã trừ phí xử lý giao dịch trước khi ghi có, trong khi hệ thống billing nội bộ ghi nhận theo giá trị gộp trước phí — đối soát phải tách rõ hai luồng số liệu này để không nhầm chênh lệch phí xử lý thành sai lệch thực sự cần điều tra.
- Khi múi giờ đóng chu kỳ billing nội bộ không trùng khớp với thời điểm processor tổng hợp báo cáo theo lịch của họ, một số giao dịch cận ranh giới ngày có thể bị xếp lệch chu kỳ giữa hai bên — cần định nghĩa rõ quy tắc xếp chu kỳ thống nhất và có bước xử lý riêng cho các giao dịch cận biên này thay vì báo sai lệch giả mỗi tháng.

---

# ERD — Base (trước khi có đối soát billing với processor)

Đây là **base**: mô hình dữ liệu suy luận cho SaaS thuê bao *trước khi* có flow đối soát. Đề bài giả định hệ thống đã có billing nội bộ phát sinh invoice (kể cả invoice điều chỉnh proration khi đổi gói) và đã tích hợp payment processor để thu tiền — nếu không có sẵn `INVOICE` và `PROCESSOR_CHARGE` thì "đối soát billing nội bộ với số tiền processor báo cáo" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ billing + thu tiền, không suy diễn thêm các module không liên quan (support ticket, usage metering chi tiết...).

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--o{ INVOICE : generates
    INVOICE }o--o| PROCESSOR_CHARGE : "collected via"

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
        date current_period_start
        date current_period_end
    }

    INVOICE {
        string invoice_id PK
        string subscription_id FK
        string type
        decimal amount
        string currency
        string status
        datetime created_at
    }

    PROCESSOR_CHARGE {
        string charge_id PK
        string invoice_id FK
        string processor_reference_id
        decimal amount_charged
        string status
        datetime charged_at
    }
```

---

# Sequence Diagram — Base: Charge Subscription Cycle

Đây là **base**, flow "thu tiền định kỳ hàng tháng" — tiền đề bắt buộc cho đối soát: chính flow này tạo ra `INVOICE` (type=recurring) và `PROCESSOR_CHARGE` tương ứng, hai dữ liệu mà đối soát sau này phải khớp với nhau theo từng chu kỳ.

```mermaid
sequenceDiagram
    participant Scheduler as Billing Scheduler
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)

    Scheduler->>Billing: Trigger billing for subscriptions due today
    Billing->>Billing: Create INVOICE (type=recurring, amount=plan.monthly_price)
    Billing->>Processor: Charge customer (invoice_id, amount)

    Processor-->>Billing: Charge result (processor_reference_id, status)
    Billing->>Billing: Create PROCESSOR_CHARGE from result
    Billing->>Billing: Update INVOICE.status based on charge result

    Billing-->>Scheduler: Cycle billing completed

    Note over Billing,Processor: No systematic check yet that invoice totals for the period
    Note over Billing,Processor: actually match what the processor collected
```

---

<a id="3-base-change-plan-proration-sqd"></a>

# Sequence Diagram — Base: Change Plan With Proration

Đây là **base**, flow "đổi gói giữa chu kỳ" — cũng là tiền đề cho đối soát vì đề bài mô tả đây là nguồn phát sinh "nhiều invoice điều chỉnh nhỏ trong cùng khoảng thời gian ngắn" mà đối soát phải khớp tổng lại với đúng một lượt thu tiền từ processor.

```mermaid
sequenceDiagram
    actor Customer
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)

    Customer->>Billing: Request plan change (new_plan_id)
    Billing->>Billing: Compute unused portion of old plan (credit)
    Billing->>Billing: Create INVOICE (type=proration_credit, negative amount)
    Billing->>Billing: Compute prorated cost of new plan
    Billing->>Billing: Create INVOICE (type=proration_charge, positive amount)

    Billing->>Processor: Charge customer net amount (sum of proration invoices)
    Processor-->>Billing: Charge result (processor_reference_id, status)
    Billing->>Billing: Create PROCESSOR_CHARGE, linked conceptually to both proration invoices

    Billing->>Billing: Update SUBSCRIPTION.plan_id
    Billing-->>Customer: Plan changed, prorated amount charged
```

---

# ERD — Enhance (sau khi có đối soát billing với processor)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — khớp nhóm invoice proration với đúng một lượt thu, phân biệt "đang retry" với "thất bại hẳn" ở dunning, cập nhật lại đối soát khi có hoàn tiền sau khớp, tách phí xử lý khỏi số tiền gộp, và xử lý riêng giao dịch cận ranh giới chu kỳ. So với base, `PROCESSOR_CHARGE` nay lưu thêm `processing_fee` và `net_amount`, và có thêm `DUNNING_ATTEMPT` để theo dõi lịch sử retry thay vì chỉ một status tĩnh.

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--o{ INVOICE : generates
    INVOICE }o--o| PROCESSOR_CHARGE : "collected via"
    PROCESSOR_CHARGE ||--o{ DUNNING_ATTEMPT : "may retry via"

    INVOICE ||--o| REFUND_EVENT : "may have"
    REFUND_EVENT |o--o| RECONCILIATION_DISCREPANCY : "re-opens"

    RECONCILIATION_RUN ||--o{ RECONCILIATION_MATCH : produces
    RECONCILIATION_MATCH ||--o{ INVOICE : groups
    RECONCILIATION_MATCH |o--o| PROCESSOR_CHARGE : "matches (optional)"
    RECONCILIATION_MATCH ||--o| RECONCILIATION_DISCREPANCY : "may yield"

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
        date current_period_start
        date current_period_end
    }

    INVOICE {
        string invoice_id PK
        string subscription_id FK
        string type
        decimal amount
        string currency
        string status
        datetime created_at
    }

    PROCESSOR_CHARGE {
        string charge_id PK
        string processor_reference_id
        decimal gross_amount
        decimal processing_fee
        decimal net_amount
        string status
        datetime charged_at
        datetime settled_at
    }

    DUNNING_ATTEMPT {
        string attempt_id PK
        string charge_id FK
        int attempt_number
        string result
        datetime attempted_at
        datetime next_retry_at
    }

    REFUND_EVENT {
        string refund_event_id PK
        string invoice_id FK
        decimal amount
        datetime occurred_at
    }

    RECONCILIATION_RUN {
        string run_id PK
        date cycle_period_start
        date cycle_period_end
        string status
    }

    RECONCILIATION_MATCH {
        string match_id PK
        string run_id FK
        string charge_id FK
        decimal invoice_total
        decimal charge_net_amount
        string match_status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string match_id FK
        string type
        decimal amount_diff
        string status
    }
```

---

<a id="5-enhance-reconcile-billing-cycle-sqd"></a>

# Sequence Diagram — Enhance: Reconcile Billing Cycle

Đây là **enhance**, flow chính hoàn toàn mới: đối soát giữa nhóm `INVOICE` (kể cả các invoice proration nhỏ phát sinh khi đổi gói, xem [3-base-change-plan-proration-sqd.md](#3-base-change-plan-proration-sqd)) với đúng một `PROCESSOR_CHARGE`. Flow thể hiện việc tách `processing_fee` khỏi `gross_amount` để so đúng `net_amount`, phân biệt trạng thái "đang chờ retry" (dunning) với "thất bại hẳn", và có bước xử lý riêng cho giao dịch cận ranh giới chu kỳ do lệch múi giờ.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)

    Ops->>Recon: Trigger reconciliation for cycle_period
    Recon->>Billing: Group INVOICE by subscription within cycle boundary (using unified cutoff rule)

    loop for each group of invoices tied to one charge attempt
        Recon->>Billing: Sum group invoice amounts (invoice_total)
        Recon->>Processor: Look up PROCESSOR_CHARGE by processor_reference_id
        Processor-->>Recon: gross_amount, processing_fee, net_amount, status

        alt charge status = succeeded and invoice_total equals net_amount
            Recon->>Recon: Create RECONCILIATION_MATCH (match_status=matched)
        else charge status = retrying (dunning in progress)
            Recon->>Processor: Check DUNNING_ATTEMPT history
            Processor-->>Recon: next_retry_at still in the future
            Recon->>Recon: Mark group as pending, exclude from discrepancy this run
            Note over Recon: Not closed yet, still eligible for a future successful retry
        else charge status = failed permanently
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=charge_failed)
        else invoice_total does not equal net_amount but charge succeeded
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=amount_mismatch)
        else transaction near cycle boundary (ambiguous cutoff)
            Recon->>Recon: Apply boundary rule, re-classify into correct cycle before comparing
        end
    end

    Recon-->>Ops: Reconciliation summary for the cycle
```

---

# Sequence Diagram — Enhance: Handle Post-Reconciliation Refund

Đây là **enhance**, flow mới xử lý trường hợp khách hủy gói giữa chu kỳ và được hoàn lại phần tiền chưa dùng *sau khi* invoice gốc đã đối soát khớp (xem [5-enhance-reconcile-billing-cycle-sqd.md](#5-enhance-reconcile-billing-cycle-sqd)). Flow này đảm bảo yêu cầu "không để hệ thống báo cáo doanh thu vẫn tính khoản đã hoàn là doanh thu thực" bằng cách mở lại đối soát cho match liên quan.

```mermaid
sequenceDiagram
    actor Customer
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)
    participant Recon as Reconciliation Service

    Customer->>Billing: Cancel subscription mid-cycle
    Billing->>Billing: Compute unused amount to refund
    Billing->>Processor: Issue refund for unused amount

    Processor-->>Billing: Refund confirmed
    Billing->>Billing: Create REFUND_EVENT (invoice_id, amount)

    Billing-->>Recon: Notify refund on an already-reconciled INVOICE
    Recon->>Recon: Look up RECONCILIATION_MATCH containing this invoice
    Recon->>Recon: Re-open RECONCILIATION_DISCREPANCY on that match (type=refunded_after_match)

    Recon-->>Billing: Revenue report excludes refunded amount going forward
```
