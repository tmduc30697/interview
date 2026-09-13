# Job tính phí renewal tự động chạy trùng lúc khách tự đổi gói/hủy gói trên subscription SaaS

**Hệ thống:** Một SaaS B2B tính phí theo subscription hàng tháng, có job billing tự động chạy vào đúng ngày renewal của mỗi khách để charge thẻ và gia hạn, khách hàng cũng có thể tự đổi gói hoặc hủy gói qua UI vào bất kỳ thời điểm nào.

**Vai trò của flow:** Flow phải xử lý đúng khi job billing tự động renewal và hành động thủ công của khách (đổi gói/hủy) chạm vào cùng 1 subscription gần như đồng thời, tránh charge sai gói hoặc charge sau khi khách đã hủy.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: job billing bắt đầu xử lý renewal cho subscription S lúc 00:00:05 (đọc gói hiện tại là "Pro"), đúng lúc khách bấm hủy subscription lúc 00:00:06, trước khi job kịp charge thẻ — yêu cầu job phải lock (hoặc dùng version check) trên record subscription trước khi charge, nếu phát hiện trạng thái đã đổi so với lúc đọc ban đầu (đã bị hủy) thì dừng ngay lập tức, không charge, không tạo invoice mới.
- Mô tả race khi khách đổi gói (downgrade từ Pro sang Basic) đúng lúc job renewal đang giữa quá trình charge theo gói Pro (đã gọi tới cổng thanh toán, đang chờ phản hồi) — quy định rõ lần renewal này vẫn hoàn tất theo gói Pro với giá đã khóa tại thời điểm job bắt đầu xử lý, việc đổi gói mới chỉ có hiệu lực từ chu kỳ kế tiếp; không được hủy giữa chừng giao dịch đang gọi cổng thanh toán vì có thể phía cổng đã charge thành công dù hệ thống nội bộ chưa kịp ghi nhận.
- Mô tả cụ thể: khách bấm hủy subscription ngay 5 giây sau khi job billing đã charge thành công chu kỳ mới — quy định rõ chính sách hoàn tiền theo tỷ lệ hoặc không hoàn cho phần thời gian còn lại của chu kỳ, và đảm bảo trạng thái subscription cuối cùng nhất quán, không rơi vào tình trạng vừa "đã charge chu kỳ mới" vừa "đã hủy ngay lập tức" gây mâu thuẫn khi hiển thị cho khách.
- Job renewal chạy hàng loạt cho nhiều subscription trong cùng 1 batch theo ngày: đảm bảo mỗi subscription được xử lý độc lập với 1 lock/version check riêng, lỗi hoặc conflict xảy ra ở 1 subscription (do khách đang thao tác trùng lúc) không được làm chậm hoặc ảnh hưởng tới việc xử lý các subscription khác trong cùng batch.
- Đảm bảo idempotency của job renewal: nếu job bị crash giữa chừng sau khi đã charge thành công ở cổng thanh toán nhưng chưa kịp cập nhật trạng thái subscription (ví dụ crash ngay sau khi gọi cổng thanh toán, trước khi ghi nhận kết quả), lần chạy lại job cho cùng subscription trong cùng chu kỳ không được charge lần 2 — phải kiểm tra trạng thái/idempotency key trước khi gọi cổng thanh toán lại.

---

# Base ERD — Subscription billing chưa có lock/version check giữa job và thao tác khách

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có cơ chế lock/version check và idempotency cho job renewal. Đề bài nói tới SaaS B2B tính phí theo subscription hàng tháng, có job billing tự động và khách tự đổi/hủy gói qua UI — nên base cần đủ: khách hàng, gói dịch vụ, subscription gắn 1 gói hiện tại, và hoá đơn mỗi lần charge thành công. Base **chưa có** version/lock trên subscription, chưa có idempotency key cho lần charge, chưa có cơ chế gói chờ hiệu lực chu kỳ sau, chưa có chính sách hoàn tiền theo tỷ lệ — những thứ đó là phần enhance.

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : owns
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--o{ INVOICE : "billed via"

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
        string status "active | cancelled"
        date current_period_start
        date current_period_end
    }
    INVOICE {
        string id PK
        string subscription_id FK
        string plan_id FK
        decimal amount
        string status "paid | failed"
        datetime billed_at
    }
```

---

# Base sequence — Job renewal charge thẻ (không lock, không version check)

Đây là **base**, flow "Job billing tự động renewal" ở trạng thái hiện tại — job đọc gói hiện tại của subscription, gọi cổng thanh toán charge theo gói đó, rồi ghi invoice và cập nhật chu kỳ mới, không kiểm tra lại subscription có bị đổi/hủy giữa chừng hay không, không có idempotency key. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu 1, 2 và 5 của đề bài đều nhằm sửa đúng các lỗ hổng race và crash-recovery ở đây.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as SUBSCRIPTION + INVOICE store
    participant Gateway as Payment Gateway

    Job->>DB: Đọc SUBSCRIPTION S (00:00:05, plan=Pro, status=active)
    Note over Job: Không lock, không ghi nhận version tại thời điểm đọc

    Job->>Gateway: Gọi charge thẻ theo giá gói Pro
    Gateway-->>Job: Charge thành công

    Job->>DB: Ghi INVOICE(plan=Pro, status=paid), UPDATE SUBSCRIPTION current_period tiếp theo

    Note over Job,DB: Nếu khách bấm hủy subscription lúc 00:00:06, ngay giữa lúc job đang gọi cổng thanh toán, job vẫn không biết và vẫn hoàn tất charge + tạo invoice cho gói đã bị hủy
```

---

# Base sequence — Khách tự đổi gói/hủy gói qua UI (ghi đè trực tiếp)

Đây là **base**, flow "Khách đổi gói/hủy gói" ở trạng thái hiện tại — khách thao tác qua UI, hệ thống ghi đè thẳng `plan_id`/`status` của subscription ngay lập tức, không biết và không quan tâm job renewal có đang xử lý song song subscription này hay không. Flow này liên quan mật thiết tới enhance vì đây chính là phía còn lại của race điều kiện mà đề bài mô tả ở yêu cầu 1 và 2.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Subscription Service
    participant DB as SUBSCRIPTION store

    Customer->>App: Bấm "Hủy subscription" (hoặc đổi gói Pro sang Basic)
    App->>DB: UPDATE SUBSCRIPTION SET status=cancelled (hoặc plan_id=Basic) ngay lập tức
    App-->>Customer: "Đã hủy thành công" (hoặc "Đã đổi gói thành công")

    Note over App,DB: Không có bước nào kiểm tra job renewal có đang giữa chừng charge thẻ cho subscription này hay không, dẫn tới 2 kịch bản race ở yêu cầu 1 và 2 của đề bài
```

---

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

---

# Enhance sequence — Job renewal với version lock, dừng ngay nếu đã bị hủy

Đây là **enhance**, flow "Job billing tự động renewal" sau khi có version check trước khi charge. So với base, flow này thay đổi ở chỗ: job ghi nhận `version` tại thời điểm đọc subscription, và ngay trước khi gọi cổng thanh toán, job kiểm tra lại version/status 1 lần nữa — nếu subscription đã bị hủy (version thay đổi) thì dừng ngay, không charge, không tạo invoice. Nếu subscription vẫn active tại thời điểm khóa, job tiếp tục charge theo giá đã khóa dù ngay sau đó khách có đổi gói, việc đổi gói chỉ ghi vào `pending_plan_id` để có hiệu lực từ chu kỳ kế tiếp.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as SUBSCRIPTION + INVOICE store
    participant Gateway as Payment Gateway

    Job->>DB: Đọc SUBSCRIPTION S lúc 00:00:05 (plan=Pro, status=active, version=7)
    Job->>DB: UPDATE RENEWAL_JOB_ITEM SET status=locked, subscription_version_at_start=7

    Note over Job,DB: Khách bấm hủy subscription lúc 00:00:06, UPDATE SUBSCRIPTION SET status=cancelled, version=8

    Job->>DB: Kiểm tra lại SUBSCRIPTION.version ngay trước khi charge (đọc version=8, khác version=7 đã khóa)

    alt Version đã đổi (subscription bị hủy giữa chừng)
        Job->>DB: UPDATE RENEWAL_JOB_ITEM SET status=skipped_cancelled
        Job->>Job: Dừng ngay, không gọi cổng thanh toán, không tạo invoice
    else Version vẫn khớp (subscription chưa bị đổi)
        Job->>Gateway: Gọi charge thẻ theo giá gói Pro đã khóa tại thời điểm job bắt đầu
        Gateway-->>Job: Charge thành công
        Job->>DB: Ghi INVOICE(plan=Pro, idempotency_key=S-2026-09, status=paid), UPDATE SUBSCRIPTION current_period tiếp theo, RENEWAL_JOB_ITEM status=success

        Note over Job,DB: Nếu ngay lúc này khách bấm đổi gói Pro sang Basic, giao dịch charge đang gọi cổng thanh toán vẫn hoàn tất theo Pro, đổi gói chỉ ghi vào pending_plan_id=Basic có hiệu lực từ chu kỳ kế tiếp, không hủy giữa chừng request đang chờ phản hồi cổng thanh toán
    end
```

---

# Enhance sequence — Đổi gói/hủy gói biết trạng thái job renewal đang xử lý

Đây là **enhance**, flow "Khách đổi gói/hủy gói" sau khi phối hợp với version lock của job renewal. So với base (ghi đè trực tiếp không kiểm tra gì), flow này thay đổi ở chỗ: mọi thao tác đổi gói/hủy đều tăng `version` của subscription bất kể job có đang xử lý hay không (đây chính là tín hiệu để job phát hiện thay đổi ở file `5-enhance-renewal-billing-job-sqd.md`); riêng đổi gói (không phải hủy) khi job đang trong lúc `charging` sẽ ghi vào `pending_plan_id` thay vì đổi `plan_id` ngay, để tránh xung đột với giao dịch đang gọi cổng thanh toán.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Subscription Service
    participant DB as SUBSCRIPTION + RENEWAL_JOB_ITEM store

    Customer->>App: Bấm hủy subscription S
    App->>DB: Đọc RENEWAL_JOB_ITEM hiện tại của S (nếu có job đang chạy trong ngày)

    alt Job chưa lock hoặc chưa bắt đầu charge (status=pending, hoặc không có job chạy hôm nay)
        App->>DB: UPDATE SUBSCRIPTION SET status=cancelled, version=version+1
        App-->>Customer: "Đã hủy thành công"
    else Job đang locked/charging subscription này
        App->>DB: UPDATE SUBSCRIPTION SET status=cancelled, version=version+1 (vẫn ghi ngay, để job tự phát hiện version đổi và tự dừng ở bước kiểm tra trước khi charge)
        App-->>Customer: "Đã ghi nhận yêu cầu hủy"
    end

    Customer->>App: Bấm đổi gói Pro sang Basic
    App->>DB: Đọc RENEWAL_JOB_ITEM hiện tại của subscription

    alt Job không đang charging (chưa chạy, hoặc đã success/skipped)
        App->>DB: UPDATE SUBSCRIPTION SET plan_id=Basic, version=version+1
        App-->>Customer: "Đã đổi gói thành công, có hiệu lực ngay"
    else Job đang charging (đã gọi cổng thanh toán theo giá Pro, đang chờ phản hồi)
        App->>DB: UPDATE SUBSCRIPTION SET pending_plan_id=Basic, version=version+1 (không đổi plan_id ngay)
        App-->>Customer: "Đã ghi nhận đổi gói, có hiệu lực từ chu kỳ kế tiếp vì chu kỳ này đang được tính phí theo gói Pro"
    end
```

---

# Enhance sequence — Hủy ngay sau khi charge thành công, áp chính sách hoàn tiền

Đây là **enhance**, flow mới phát sinh từ enhance — khách bấm hủy subscription 5 giây sau khi job billing đã charge thành công chu kỳ mới. Xử lý đúng yêu cầu 3 của đề bài: áp chính sách hoàn tiền theo tỷ lệ (hoặc không hoàn) cho phần thời gian còn lại của chu kỳ, và đảm bảo trạng thái subscription cuối cùng nhất quán, không mâu thuẫn giữa "đã charge chu kỳ mới" và "đã hủy ngay lập tức".

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as SUBSCRIPTION + INVOICE store
    actor Customer
    participant App as Subscription Service
    participant Refund as REFUND store

    Job->>DB: Charge thành công chu kỳ mới, ghi INVOICE(status=paid), UPDATE SUBSCRIPTION current_period_start=hôm nay, current_period_end=+30 ngày

    Note over Customer,App: 5 giây sau

    Customer->>App: Bấm hủy subscription
    App->>DB: Đọc SUBSCRIPTION, thấy INVOICE mới nhất status=paid, billed_at cách đây 5 giây

    App->>App: Áp chính sách hoàn tiền theo tỷ lệ cho phần thời gian còn lại của chu kỳ vừa charge (gần như toàn bộ 30 ngày)
    App->>Refund: Ghi REFUND(invoice_id, refund_type=prorated, amount=tính theo số ngày còn lại)
    App->>DB: UPDATE SUBSCRIPTION SET status=cancelled, current_period_end=hôm nay (kết thúc chu kỳ ngay, không giữ hiệu lực tới hết 30 ngày)

    App-->>Customer: "Đã hủy, đã hoàn tiền theo tỷ lệ cho phần thời gian chưa dùng"

    Note over DB,Refund: Trạng thái cuối cùng nhất quán: subscription = cancelled, invoice chu kỳ mới vẫn giữ status=paid (đúng sự thật đã charge), phần chênh lệch được thể hiện qua REFUND thay vì sửa lại INVOICE
```

---

# Enhance sequence — Batch renewal xử lý độc lập, lỗi 1 subscription không lan sang subscription khác

Đây là **enhance**, flow mới phát sinh từ enhance — job renewal chạy hàng loạt cho nhiều subscription đến hạn trong ngày, mỗi subscription là 1 `RENEWAL_JOB_ITEM` độc lập với version lock riêng theo đúng flow ở file `5-enhance-renewal-billing-job-sqd.md`. Xử lý đúng yêu cầu 4 của đề bài: conflict hoặc lỗi ở 1 subscription không làm chậm hoặc ảnh hưởng tới việc xử lý các subscription khác trong cùng batch.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as RENEWAL_JOB_ITEM store
    participant GatewayA as Payment Gateway (sub A)
    participant GatewayC as Payment Gateway (sub C)

    Job->>DB: Tạo RENEWAL_JOB_RUN cho batch hôm nay, liệt kê các subscription đến hạn A, B, C

    par Xử lý subscription A độc lập
        Job->>DB: Lock A (version check), gọi GatewayA
        GatewayA-->>Job: Charge thành công
        Job->>DB: UPDATE RENEWAL_JOB_ITEM(A) status=success
    and Xử lý subscription B độc lập
        Job->>DB: Lock B (version check), phát hiện version đã đổi do khách vừa hủy
        Job->>DB: UPDATE RENEWAL_JOB_ITEM(B) status=skipped_cancelled
    and Xử lý subscription C độc lập
        Job->>DB: Lock C (version check), gọi GatewayC
        GatewayC-->>Job: Timeout/lỗi
        Job->>DB: UPDATE RENEWAL_JOB_ITEM(C) status=failed, ghi lại để retry riêng
    end

    Note over Job,DB: Subscription B bị skip và C bị lỗi không làm chậm hay chặn việc subscription A vẫn xử lý thành công đúng hạn, vì mỗi item có lock/version check và trạng thái riêng biệt
```

---

# Enhance sequence — Job crash giữa chừng, restart không charge lần 2

Đây là **enhance**, flow mới phát sinh từ enhance — xử lý đúng yêu cầu 5 của đề bài: job crash ngay sau khi đã charge thành công ở cổng thanh toán nhưng chưa kịp ghi nhận kết quả vào `INVOICE`, lần chạy lại cho cùng subscription trong cùng chu kỳ phải kiểm tra idempotency trước khi gọi cổng thanh toán lại, tránh charge lần 2.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job (lần chạy 1)
    participant Gateway as Payment Gateway
    participant DB as INVOICE store
    participant JobRetry as Renewal Billing Job (lần chạy lại)

    Job->>DB: Kiểm tra INVOICE với idempotency_key=S-2026-09, chưa tồn tại
    Job->>Gateway: Gọi charge thẻ subscription S theo giá gói Pro
    Gateway-->>Job: Charge thành công (nhưng job crash ngay sau khi nhận response, trước khi ghi INVOICE)

    Note over Job,DB: Job process chết đột ngột, chưa kịp UPDATE INVOICE/SUBSCRIPTION

    Note over JobRetry: Job được lên lịch chạy lại (do batch chưa hoàn tất hoặc bị retry tự động)

    JobRetry->>DB: Kiểm tra INVOICE với idempotency_key=S-2026-09 trước khi gọi cổng thanh toán

    alt idempotency_key đã tồn tại với status=paid
        DB-->>JobRetry: Đã có INVOICE(status=paid) từ lần chạy trước
        JobRetry->>JobRetry: Không gọi lại cổng thanh toán, chỉ đảm bảo SUBSCRIPTION.current_period được cập nhật đúng nếu lần trước cũng chưa kịp ghi
    else idempotency_key chưa tồn tại hoặc status=failed
        JobRetry->>Gateway: Query trạng thái giao dịch trước đó theo idempotency_key (nếu gateway hỗ trợ), hoặc thử charge với cùng idempotency_key để gateway tự dedupe
        Gateway-->>JobRetry: Trả về kết quả thật (đã charge thành công trước đó)
        JobRetry->>DB: Ghi INVOICE(idempotency_key=S-2026-09, status=paid) đúng 1 lần
    end

    Note over JobRetry,DB: Nhờ kiểm tra idempotency_key trước khi charge, khách không bị charge 2 lần dù job restart giữa chừng sau crash
```
