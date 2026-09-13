# Marketplace đa seller đối soát payout cho người bán

**Hệ thống:** Một marketplace đa seller, sàn thu tiền từ người mua rồi định kỳ trả (payout) cho từng seller sau khi trừ phí hoa hồng sàn và các khoản hoàn tiền phát sinh.

**Vai trò của flow:** Đối soát giữa số tiền hệ thống nội bộ tính seller được nhận (sau phí, sau hoàn tiền) và số tiền thực tế đã được chuyển ra ngoài qua cổng thanh toán cho từng seller.

**Yêu cầu cụ thể:**
- Một seller có thể có hàng trăm đơn hàng hoàn tất trong kỳ, một số đơn bị hoàn tiền/khiếu nại xử lý sau khi đã tính vào kỳ payout — cần xác định rõ mốc thời gian cắt (cutoff) nhất quán cho từng kỳ, tránh trường hợp đơn vừa hoàn tất bị tính hai lần vào hai kỳ khác nhau hoặc bị bỏ sót không kỳ nào tính tới.
- Khi một đơn hàng đã được tính vào payout đã chuyển tiền cho seller, nhưng sau đó phát sinh hoàn tiền cho người mua, hệ thống phải xử lý đúng việc thu hồi/trừ vào kỳ payout tiếp theo của seller đó — đối soát phải phát hiện được các khoản âm này thay vì chỉ xử lý cộng dồn một chiều.
- Lệnh chuyển tiền ra ngoài cho seller qua cổng thanh toán có thể ở trạng thái pending/thất bại/bị trả lại (ví dụ tài khoản ngân hàng seller sai thông tin) — đối soát phải phân biệt rõ "đã lên lệnh chuyển" và "seller thực nhận được tiền", không đóng kỳ payout khi còn lệnh chuyển chưa xác nhận thành công.
- Phí hoa hồng sàn được tính tại thời điểm đơn hàng hoàn tất có thể khác với mức phí áp dụng tại thời điểm đối soát nếu chính sách phí thay đổi giữa chừng — đối soát phải dùng đúng mức phí đã chốt tại thời điểm phát sinh giao dịch, không tính lại theo cấu hình phí hiện tại gây sai lệch số liệu lịch sử.
- Khi phát hiện sai lệch giữa số tiền sàn tính seller được nhận và số tiền thực đã chuyển, cần có cơ chế giữ (hold) kỳ payout tiếp theo của seller đó lại để điều chỉnh, thay vì tiếp tục chuyển tiền theo số liệu có khả năng sai, đồng thời không làm ảnh hưởng payout của các seller khác không liên quan.

---

# ERD — Base (trước khi có đối soát payout seller)

Đây là **base**: mô hình dữ liệu suy luận cho marketplace đa seller *trước khi* có flow đối soát payout. Đề bài giả định sàn đã thu tiền từ người mua, đã tính hoa hồng theo cấu hình phí, và đã có cơ chế tính/chuyển tiền định kỳ cho seller — nếu không có sẵn `ORDER`, `COMMISSION_CONFIG` và `PAYOUT` thì "đối soát số tiền tính toán với số tiền thực chuyển" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ payout, không suy diễn thêm các module không liên quan (catalog, review...).

```mermaid
erDiagram
    SELLER ||--o{ ORDER : fulfills
    COMMISSION_CONFIG ||--o{ ORDER : "applies to"
    SELLER ||--o{ PAYOUT : receives
    PAYOUT ||--o{ ORDER : aggregates
    PAYOUT ||--o| PAYOUT_TRANSFER : "sent via"

    SELLER {
        string seller_id PK
        string business_name
        string bank_account_info
        string status
    }

    ORDER {
        string order_id PK
        string seller_id FK
        decimal order_amount
        decimal commission_amount
        string status
        datetime completed_at
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    PAYOUT {
        string payout_id PK
        string seller_id FK
        date period_start
        date period_end
        decimal total_amount
        string status
    }

    PAYOUT_TRANSFER {
        string transfer_id PK
        string payout_id FK
        decimal amount
        string gateway_reference_code
        string status
        datetime initiated_at
    }
```

---

<a id="2-base-calculate-payout-sqd"></a>

# Sequence Diagram — Base: Calculate Payout

Đây là **base**, flow "tính và chuyển payout cho seller" theo kỳ — tiền đề bắt buộc cho đối soát: chính flow này tạo ra `PAYOUT` (số tiền sàn tính seller được nhận) và `PAYOUT_TRANSFER` (lệnh chuyển tiền thực), hai con số mà đối soát sau này phải khớp với nhau. Ở base, flow này còn đơn giản, chưa xử lý cutoff nhất quán hay lịch sử phí.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Payout as Payout Service
    participant Order as Order Service
    participant Gateway as Payment Gateway (Partner)

    Ops->>Payout: Trigger payout run for period
    Payout->>Order: List completed ORDER for each seller in period
    Order-->>Payout: Orders with order_amount, commission_amount

    Payout->>Payout: Sum orders, apply current COMMISSION_CONFIG rate
    Payout->>Payout: Create PAYOUT (total_amount) per seller

    Payout->>Gateway: Initiate transfer (amount, seller bank info)
    Gateway-->>Payout: Transfer accepted (gateway_reference_code)
    Payout->>Payout: Create PAYOUT_TRANSFER (status=pending)

    Payout-->>Ops: Payout run summary

    Note over Payout,Gateway: No confirmation yet that seller actually received the money
    Note over Payout,Gateway: and no check against what the transfer partner actually moved
```

---

# ERD — Enhance (sau khi có đối soát payout seller)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — cutoff nhất quán theo kỳ, xử lý hoàn tiền phát sinh sau payout (clawback), phân biệt "đã lên lệnh chuyển" với "seller thực nhận", và hold kỳ payout tiếp theo khi phát hiện sai lệch. So với base, `ORDER` nay lưu thêm `commission_rate_snapshot` chốt tại thời điểm hoàn tất (không tính lại theo cấu hình hiện tại), và `PAYOUT_TRANSFER` có thêm trạng thái xác nhận rõ ràng.

```mermaid
erDiagram
    SELLER ||--o{ ORDER : fulfills
    COMMISSION_CONFIG ||--o{ ORDER : "applies to"
    SELLER ||--o{ PAYOUT : receives
    PAYOUT ||--o{ ORDER : aggregates
    PAYOUT ||--o| PAYOUT_TRANSFER : "sent via"

    PAYOUT_PERIOD ||--o{ PAYOUT : defines
    ORDER ||--o{ REFUND_EVENT : "may have"
    REFUND_EVENT |o--o| PAYOUT_CLAWBACK : "triggers"
    PAYOUT_CLAWBACK |o--o| PAYOUT : "deducted in next"

    RECONCILIATION_RUN ||--o{ RECONCILIATION_DISCREPANCY : yields
    RECONCILIATION_DISCREPANCY |o--o| PAYOUT : "relates to"
    RECONCILIATION_DISCREPANCY |o--o| PAYOUT_HOLD : "may trigger"
    PAYOUT_HOLD |o--o| PAYOUT : "holds next"

    SELLER {
        string seller_id PK
        string business_name
        string bank_account_info
        string status
    }

    ORDER {
        string order_id PK
        string seller_id FK
        decimal order_amount
        decimal commission_amount
        decimal commission_rate_snapshot
        string status
        datetime completed_at
        string payout_id FK
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    PAYOUT_PERIOD {
        string period_id PK
        date period_start
        date period_end
        datetime cutoff_at
        string status
    }

    PAYOUT {
        string payout_id PK
        string seller_id FK
        string period_id FK
        decimal total_amount
        string status
    }

    PAYOUT_TRANSFER {
        string transfer_id PK
        string payout_id FK
        decimal amount
        string gateway_reference_code
        string status
        datetime initiated_at
        datetime confirmed_at
    }

    REFUND_EVENT {
        string refund_event_id PK
        string order_id FK
        decimal amount
        datetime occurred_at
    }

    PAYOUT_CLAWBACK {
        string clawback_id PK
        string refund_event_id FK
        string applied_to_payout_id FK
        decimal amount
        string status
    }

    RECONCILIATION_RUN {
        string run_id PK
        string period_id FK
        datetime started_at
        datetime completed_at
        string status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string run_id FK
        string payout_id FK
        decimal amount_diff
        string type
        string status
    }

    PAYOUT_HOLD {
        string hold_id PK
        string seller_id FK
        string discrepancy_id FK
        string held_payout_id FK
        string reason
        string status
        datetime created_at
        datetime released_at
    }
```

---

# Sequence Diagram — Enhance: Calculate Payout

Đây là **enhance**, flow "tính payout" đã có ở base ([2-base-calculate-payout-sqd.md](#2-base-calculate-payout-sqd)) nhưng nay thay đổi ở hai điểm: (1) dùng `PAYOUT_PERIOD.cutoff_at` cố định để xác định đơn nào thuộc kỳ nào thay vì mốc thời gian ngầm định, tránh tính trùng/bỏ sót; (2) mỗi `ORDER` dùng đúng `commission_rate_snapshot` đã chốt tại thời điểm hoàn tất thay vì `COMMISSION_CONFIG` hiện hành.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Payout as Payout Service
    participant Order as Order Service
    participant Hold as Payout Hold Service
    participant Gateway as Payment Gateway (Partner)

    Ops->>Payout: Trigger payout run for PAYOUT_PERIOD (cutoff_at)
    Payout->>Payout: Lock cutoff_at, only orders completed_at < cutoff_at are eligible

    Payout->>Hold: Check for active PAYOUT_HOLD on any seller
    Hold-->>Payout: List of held sellers, excluded from this run

    Payout->>Order: List eligible completed ORDER per seller (using commission_rate_snapshot)
    Order-->>Payout: Orders with order_amount, commission_amount already snapshotted

    Payout->>Payout: Sum orders per seller, subtract pending PAYOUT_CLAWBACK if any
    Payout->>Payout: Create PAYOUT (total_amount) per non-held seller

    Payout->>Gateway: Initiate transfer (amount, seller bank info)
    Gateway-->>Payout: Transfer accepted (gateway_reference_code)
    Payout->>Payout: Create PAYOUT_TRANSFER (status=pending)

    Payout-->>Ops: Payout run summary, held sellers listed separately
```

---

# Sequence Diagram — Enhance: Reconcile Payout

Đây là **enhance**, flow mới hoàn toàn: đối soát giữa `PAYOUT.total_amount` (số tiền sàn tính) và trạng thái thực tế của `PAYOUT_TRANSFER`. Flow này thể hiện yêu cầu phân biệt "đã lên lệnh chuyển" với "seller thực nhận" — chỉ coi kỳ đã đóng khi transfer được đối tác xác nhận thành công — và tạo `PAYOUT_HOLD` cho seller có sai lệch mà không ảnh hưởng các seller khác.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Payout as Payout Service
    participant Gateway as Payment Gateway (Partner)
    participant Hold as Payout Hold Service

    Recon->>Payout: List PAYOUT_TRANSFER for the period, status=pending
    Recon->>Gateway: Query actual transfer status by gateway_reference_code

    loop for each PAYOUT_TRANSFER
        Gateway-->>Recon: Transfer status (confirmed/failed/returned) and actual amount
        alt confirmed and amount matches
            Recon->>Payout: Mark PAYOUT_TRANSFER confirmed_at
            Recon->>Recon: Mark PAYOUT as reconciled
        else failed or returned, or amount mismatch
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (payout_id, amount_diff, type)
            Recon->>Hold: Create PAYOUT_HOLD for this seller's next PAYOUT
            Note over Hold: Only this seller's next payout is held
            Note over Hold: other sellers in the same run are unaffected
        end
    end

    Recon-->>Ops: Reconciliation summary, discrepancies and holds created
```

---

# Sequence Diagram — Enhance: Handle Refund Clawback

Đây là **enhance**, flow mới xử lý trường hợp một `ORDER` đã được tính vào `PAYOUT` đã chuyển tiền, nhưng sau đó phát sinh hoàn tiền cho người mua. Flow này thể hiện yêu cầu "phát hiện được các khoản âm" thay vì chỉ cộng dồn một chiều, bằng cách tạo `PAYOUT_CLAWBACK` để trừ vào kỳ payout tiếp theo của đúng seller đó.

```mermaid
sequenceDiagram
    participant Buyer
    participant Order as Order Service
    participant Payout as Payout Service

    Buyer->>Order: Request refund for a completed order
    Order->>Order: Process refund, create REFUND_EVENT
    Order-->>Payout: Notify refund on ORDER already included in a past PAYOUT

    Payout->>Payout: Look up ORDER.payout_id, confirm it was already paid out
    Payout->>Payout: Create PAYOUT_CLAWBACK (refund_event_id, amount, status=pending)

    Note over Payout: Clawback is not deducted retroactively from the past payout
    Note over Payout: it is applied as a negative line item in the seller's next PAYOUT_PERIOD

    Payout-->>Order: Clawback recorded, will offset next payout
```
