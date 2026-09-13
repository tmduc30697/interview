# Ứng dụng gọi xe đối soát cuốc xe, tiền tài xế và hoa hồng

**Hệ thống:** Một ứng dụng gọi xe, khách đặt xe, tài xế hoàn tất cuốc, nền tảng thu tiền và chia lại cho tài xế sau khi trừ hoa hồng, một số cuốc bị hủy giữa chừng hoặc phát sinh tranh chấp giá.

**Vai trò của flow:** Đối soát giữa số tiền khách trả, số tiền tài xế được nhận, và hoa hồng nền tảng giữ lại, đảm bảo ba khoản này khớp đúng cho từng cuốc xe và không có cuốc nào bị tính sai/bỏ sót.

**Yêu cầu cụ thể:**
- Cuốc xe bị hủy giữa chừng sau khi tài xế đã di chuyển một đoạn có thể phát sinh phí hủy một phần cho tài xế nhưng không phải giá cuốc đầy đủ — đối soát phải phân biệt rõ luồng tiền của cuốc hoàn tất và cuốc hủy có phí bồi thường, không gộp chung logic tính hoa hồng như cuốc bình thường.
- Giá cuốc thực tế cuối cùng có thể khác giá ước tính ban đầu hiển thị cho khách (do đổi lộ trình giữa đường, chờ đợi phát sinh, tranh chấp về quãng đường) — đối soát phải dùng đúng giá đã chốt cuối cùng làm cơ sở tính hoa hồng và tiền tài xế, xử lý được trường hợp giá bị điều chỉnh lại sau khi cuốc đã kết thúc và tiền đã tạm ghi nhận.
- Tài xế nhận tiền mặt trực tiếp từ khách cho một số cuốc trong khi hoa hồng nền tảng vẫn phải được thu lại từ tài xế theo cách khác (trừ vào ví/đợt thanh toán sau) — đối soát phải theo dõi riêng luồng tiền mặt này để không nhầm là "tiền chưa về" khi thực chất khách đã trả trực tiếp cho tài xế.
- Khi có tranh chấp giá giữa khách và tài xế được xử lý sau khi cuốc đã kết thúc và tiền đã được chia, khoản điều chỉnh phát sinh sau phải được đối soát lại đúng cho cả ba bên mà không làm sai lệch báo cáo hoa hồng của các cuốc khác không liên quan trong cùng kỳ.
- Với tài xế có khối lượng cuốc lớn mỗi ngày, tổng tiền tài xế nhận được tổng hợp theo đợt thanh toán định kỳ phải khớp chính xác với tổng từng cuốc riêng lẻ cộng lại trừ các khoản điều chỉnh phát sinh trong kỳ — cần cơ chế đối soát ở cả mức chi tiết từng cuốc và mức tổng hợp theo đợt để phát hiện sai lệch dù nhỏ không bị pha loãng/che khuất khi chỉ nhìn tổng số cuối kỳ.

---

# ERD — Base (trước khi có đối soát cuốc xe/tài xế/hoa hồng)

Đây là **base**: mô hình dữ liệu suy luận cho ứng dụng gọi xe *trước khi* có flow đối soát ba chiều (khách trả - tài xế nhận - hoa hồng nền tảng). Đề bài giả định hệ thống đã có cuốc xe với giá cước, đã tính hoa hồng và đã chia tiền cho tài xế qua ví — nếu không có sẵn `TRIP`, `FARE` và `DRIVER_WALLET` thì "đối soát ba khoản khớp đúng cho từng cuốc" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ cuốc xe + chia tiền, không suy diễn thêm các module không liên quan (khuyến mãi, đánh giá tài xế...).

```mermaid
erDiagram
    RIDER ||--o{ TRIP : requests
    DRIVER ||--o{ TRIP : completes
    TRIP ||--|| FARE : has
    COMMISSION_CONFIG ||--o{ TRIP : "applies to"
    DRIVER ||--|| DRIVER_WALLET : has
    TRIP ||--o| WALLET_ENTRY : "credits/debits"
    DRIVER_WALLET ||--o{ WALLET_ENTRY : accumulates

    RIDER {
        string rider_id PK
        string full_name
        string phone
    }

    DRIVER {
        string driver_id PK
        string full_name
        string phone
        string status
    }

    TRIP {
        string trip_id PK
        string rider_id FK
        string driver_id FK
        string status
        datetime requested_at
        datetime completed_at
    }

    FARE {
        string fare_id PK
        string trip_id FK
        decimal estimated_amount
        decimal final_amount
        string currency
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    DRIVER_WALLET {
        string wallet_id PK
        string driver_id FK
        decimal balance
    }

    WALLET_ENTRY {
        string entry_id PK
        string wallet_id FK
        string trip_id FK
        decimal amount
        string entry_type
        datetime created_at
    }
```

---

<a id="2-base-complete-trip-payment-sqd"></a>

# Sequence Diagram — Base: Complete Trip and Split Payment

Đây là **base**, flow "hoàn tất cuốc xe và chia tiền" — tiền đề bắt buộc cho đối soát: chính flow này chốt `FARE.final_amount`, tính hoa hồng theo `COMMISSION_CONFIG`, và ghi `WALLET_ENTRY` cho tài xế. Ba con số (khách trả, tài xế nhận, hoa hồng giữ lại) sinh ra ở đây chính là ba khoản mà đối soát sau này phải khớp với nhau cho từng cuốc.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Driver->>Trip: Mark trip completed (final route, distance)
    Trip->>Trip: Compute FARE.final_amount from final route
    Trip->>Payment: Charge rider final_amount

    Payment-->>Trip: Payment captured
    Trip->>Trip: Apply COMMISSION_CONFIG rate to final_amount

    Trip->>Wallet: Credit driver (final_amount minus commission)
    Wallet->>Wallet: Create WALLET_ENTRY (amount, entry_type=trip_earning)
    Wallet-->>Trip: Driver wallet updated

    Trip-->>Rider: Trip receipt (final fare)
    Trip-->>Driver: Trip earning summary
```

---

# ERD — Enhance (sau khi có đối soát cuốc xe/tài xế/hoa hồng)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — cuốc bị hủy giữa chừng có phí một phần, giá cước điều chỉnh sau khi cuốc kết thúc, thanh toán tiền mặt theo dõi riêng, điều chỉnh phát sinh từ tranh chấp giá, và đối soát ở cả mức cuốc lẫn mức đợt thanh toán. So với base, `WALLET_ENTRY` nay còn được tạo bởi `FARE_ADJUSTMENT` và `TRIP_CANCELLATION` (không chỉ bởi cuốc hoàn tất bình thường), và `TRIP` có thêm cờ `payment_method` để phân biệt tiền mặt.

```mermaid
erDiagram
    RIDER ||--o{ TRIP : requests
    DRIVER ||--o{ TRIP : completes
    TRIP ||--|| FARE : has
    COMMISSION_CONFIG ||--o{ TRIP : "applies to"
    DRIVER ||--|| DRIVER_WALLET : has
    TRIP ||--o{ WALLET_ENTRY : "credits/debits"
    DRIVER_WALLET ||--o{ WALLET_ENTRY : accumulates

    TRIP |o--o| TRIP_CANCELLATION : "may end in"
    FARE ||--o{ FARE_ADJUSTMENT : "may have"
    FARE_ADJUSTMENT |o--o| WALLET_ENTRY : generates

    DRIVER_WALLET ||--o{ PAYOUT_BATCH : "settled via"
    PAYOUT_BATCH ||--o{ WALLET_ENTRY : aggregates

    RECONCILIATION_RUN ||--o{ RECONCILIATION_DISCREPANCY : yields
    RECONCILIATION_DISCREPANCY |o--o| TRIP : "relates to"
    RECONCILIATION_DISCREPANCY |o--o| PAYOUT_BATCH : "relates to"

    RIDER {
        string rider_id PK
        string full_name
        string phone
    }

    DRIVER {
        string driver_id PK
        string full_name
        string phone
        string status
    }

    TRIP {
        string trip_id PK
        string rider_id FK
        string driver_id FK
        string status
        string payment_method
        datetime requested_at
        datetime completed_at
    }

    FARE {
        string fare_id PK
        string trip_id FK
        decimal estimated_amount
        decimal final_amount
        string currency
    }

    FARE_ADJUSTMENT {
        string adjustment_id PK
        string fare_id FK
        decimal amount_diff
        string reason
        string approved_by
        datetime created_at
    }

    TRIP_CANCELLATION {
        string cancellation_id PK
        string trip_id FK
        string cancelled_by
        decimal cancellation_fee
        string reason
        datetime cancelled_at
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    DRIVER_WALLET {
        string wallet_id PK
        string driver_id FK
        decimal balance
    }

    WALLET_ENTRY {
        string entry_id PK
        string wallet_id FK
        string trip_id FK
        string adjustment_id FK
        decimal amount
        string entry_type
        datetime created_at
    }

    PAYOUT_BATCH {
        string batch_id PK
        string driver_id FK
        date period_start
        date period_end
        decimal total_amount
        string status
    }

    RECONCILIATION_RUN {
        string run_id PK
        date period_start
        date period_end
        string status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string run_id FK
        string trip_id FK
        string batch_id FK
        string level
        decimal amount_diff
        string status
    }
```

---

<a id="4-enhance-complete-trip-payment-sqd"></a>

# Sequence Diagram — Enhance: Complete Trip and Split Payment

Đây là **enhance**, flow "hoàn tất cuốc và chia tiền" đã có ở base ([2-base-complete-trip-payment-sqd.md](#2-base-complete-trip-payment-sqd)) nhưng nay thay đổi ở chỗ: cuốc trả tiền mặt không đi qua Payment Service để thu tiền từ khách, nhưng hoa hồng vẫn phải bị trừ vào ví tài xế theo hướng ngược lại, thay vì chỉ cộng dồn một chiều như base.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Driver->>Trip: Mark trip completed (final route, distance, payment_method)
    Trip->>Trip: Compute FARE.final_amount from final route

    alt payment_method = card/wallet
        Trip->>Payment: Charge rider final_amount
        Payment-->>Trip: Payment captured
        Trip->>Wallet: Credit driver (final_amount minus commission)
        Wallet->>Wallet: Create WALLET_ENTRY (entry_type=trip_earning)
    else payment_method = cash
        Note over Rider,Driver: Rider pays driver directly, cash never touches the platform
        Trip->>Wallet: Debit driver for commission only (rider already paid driver in cash)
        Wallet->>Wallet: Create WALLET_ENTRY (entry_type=cash_trip_commission, amount negative)
    end

    Wallet-->>Trip: Driver wallet updated
    Trip-->>Rider: Trip receipt (final fare)
    Trip-->>Driver: Trip earning summary
```

---

# Sequence Diagram — Enhance: Cancel Trip With Partial Fee

Đây là **enhance**, flow mới xử lý cuốc bị hủy giữa chừng sau khi tài xế đã di chuyển một đoạn. Flow này thể hiện yêu cầu "phân biệt rõ luồng tiền của cuốc hoàn tất và cuốc hủy có phí bồi thường" — không tính hoa hồng theo logic cuốc bình thường, mà tạo `TRIP_CANCELLATION` với phí hủy một phần riêng.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Rider->>Trip: Cancel trip while driver is en route
    Trip->>Trip: Compute partial cancellation_fee based on distance driven

    Trip->>Trip: Create TRIP_CANCELLATION (cancelled_by=rider, cancellation_fee)
    Trip->>Payment: Charge rider cancellation_fee only (not full fare)
    Payment-->>Trip: Payment captured

    Trip->>Wallet: Credit driver full cancellation_fee, no commission deducted
    Wallet->>Wallet: Create WALLET_ENTRY (entry_type=cancellation_compensation)
    Wallet-->>Trip: Driver wallet updated

    Trip->>Trip: Mark TRIP status=cancelled
    Trip-->>Rider: Cancellation confirmed, fee charged
    Trip-->>Driver: Compensation credited
```

---

# Sequence Diagram — Enhance: Fare Adjustment After Dispute

Đây là **enhance**, flow mới xử lý tranh chấp giá giữa khách và tài xế được giải quyết *sau khi* cuốc đã kết thúc và tiền đã chia (theo [4-enhance-complete-trip-payment-sqd.md](#4-enhance-complete-trip-payment-sqd)). Flow này thể hiện yêu cầu điều chỉnh đúng cho cả ba bên (khách, tài xế, hoa hồng) mà không làm sai lệch báo cáo của các cuốc khác trong cùng kỳ.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Ops->>Trip: Resolve fare dispute, set corrected final_amount
    Trip->>Trip: Create FARE_ADJUSTMENT (fare_id, amount_diff, reason, approved_by)

    alt corrected amount lower than original
        Trip->>Payment: Refund rider the difference
        Trip->>Wallet: Debit driver, and reverse commission proportionally
    else corrected amount higher than original
        Trip->>Payment: Charge rider the additional difference
        Trip->>Wallet: Credit driver, and collect additional commission proportionally
    end

    Wallet->>Wallet: Create WALLET_ENTRY (adjustment_id, entry_type=fare_adjustment)
    Wallet-->>Trip: Driver wallet updated

    Trip-->>Ops: Adjustment applied, this trip only
    Note over Trip: Only this TRIP's FARE and WALLET_ENTRY are touched
    Note over Trip: other trips in the same commission period stay untouched
```

---

# Sequence Diagram — Enhance: Driver Payout Reconciliation

Đây là **enhance**, flow mới đối soát ở cả mức chi tiết từng cuốc lẫn mức tổng hợp theo đợt thanh toán (`PAYOUT_BATCH`). Flow này thể hiện yêu cầu "đối soát ở cả mức chi tiết và mức tổng hợp để phát hiện sai lệch dù nhỏ không bị pha loãng" khi tài xế có khối lượng cuốc lớn mỗi ngày.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Wallet as Driver Wallet Service
    participant Payout as Payout Batch Service

    Recon->>Payout: List PAYOUT_BATCH for the period
    loop for each PAYOUT_BATCH
        Recon->>Wallet: Sum all WALLET_ENTRY (trip earnings, adjustments, cancellations) for this driver/period
        Wallet-->>Recon: Sum of individual entries

        alt batch total matches sum of entries
            Recon->>Recon: Mark PAYOUT_BATCH as reconciled
        else mismatch found
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (level=batch, batch_id, amount_diff)
        end

        Recon->>Wallet: Drill into individual TRIP-level entries for this batch
        loop for each TRIP in batch
            Recon->>Recon: Verify FARE.final_amount, commission, driver credit are all consistent for this trip
            alt trip-level inconsistency found
                Recon->>Recon: Create RECONCILIATION_DISCREPANCY (level=trip, trip_id, amount_diff)
            end
        end
    end

    Recon-->>Ops: Reconciliation summary, discrepancies at both trip and batch level
```
