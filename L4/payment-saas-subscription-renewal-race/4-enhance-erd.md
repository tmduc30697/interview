# Enhance ERD — Thêm version lock, idempotency, gói chờ hiệu lực và hoàn tiền theo tỷ lệ

Đây là **enhance**, mô hình dữ liệu sau khi áp cơ chế lock/version check + idempotency lên base. So với base: `SUBSCRIPTION` thêm `version` (optimistic lock, job kiểm tra trước khi charge) và `pending_plan_id` (đổi gói chỉ có hiệu lực từ chu kỳ kế tiếp khi job đang giữa chừng charge). `INVOICE` thêm `idempotency_key` unique theo (subscription_id, kỳ billing) để job restart sau crash không charge lần 2. Thêm mới `RENEWAL_JOB_RUN` + `RENEWAL_JOB_ITEM` — mỗi subscription trong batch được xử lý độc lập với trạng thái/lock riêng, lỗi 1 item không ảnh hưởng item khác. Thêm mới `REFUND` — ghi nhận chính sách hoàn tiền theo tỷ lệ hoặc không hoàn khi khách hủy ngay sau khi đã charge thành công.

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : owns
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--o{ INVOICE : "billed via"
    INVOICE ||--o{ REFUND : "may be refunded via"
    RENEWAL_JOB_RUN ||--o{ RENEWAL_JOB_ITEM : contains
    SUBSCRIPTION ||--o{ RENEWAL_JOB_ITEM : "processed by"

    CUSTOMER {
        string id PK
        string company_name
    }
    PLAN {
        string id PK
        string name "Basic | Pro | Enterprise"
        decimal monthly_price
    }
    SUBSCRIPTION {
        string id PK
        string customer_id FK
        string plan_id FK
        string pending_plan_id FK "gói mới chờ hiệu lực từ chu kỳ kế tiếp, null nếu không đổi gói"
        string status "active | cancelled"
        int version "optimistic lock, job kiểm tra trước khi charge"
        date current_period_start
        date current_period_end
    }
    INVOICE {
        string id PK
        string subscription_id FK
        string plan_id FK
        string idempotency_key UK "vd subscription_id + billing_period, chống charge lần 2 khi job restart"
        decimal amount
        string status "paid | failed"
        datetime billed_at
    }
    REFUND {
        string id PK
        string invoice_id FK
        string subscription_id FK
        string refund_type "prorated | none"
        decimal amount
        datetime created_at
    }
    RENEWAL_JOB_RUN {
        string id PK
        date run_date
        string batch_status "running | completed"
    }
    RENEWAL_JOB_ITEM {
        string id PK
        string job_run_id FK
        string subscription_id FK
        int subscription_version_at_start "version đọc được lúc bắt đầu xử lý item này"
        string status "pending | locked | charging | success | skipped_cancelled | failed"
        datetime started_at
        datetime completed_at
    }
```
