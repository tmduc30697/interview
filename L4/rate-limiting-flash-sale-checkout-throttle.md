# Throttling request checkout trong flash sale để bảo vệ hệ thống backend

**Hệ thống:** E-commerce cần throttle lượng request checkout đồng thời trong giờ flash sale để tránh backend (DB, payment) bị quá tải sập hoàn toàn.

**Vai trò của flow:** Giới hạn số lượng request được xử lý đồng thời ở tầng vào (ví dụ qua hàng đợi ảo/virtual waiting room) để bảo vệ hệ thống, đổi lại một số user phải chờ theo lượt.

**Yêu cầu cụ thể:**
- Khi lượng request checkout vượt khả năng xử lý an toàn của backend, hệ thống phải chuyển user vào "phòng chờ" (queue) hiển thị vị trí/thời gian ước tính, không để request dồn thẳng vào backend gây sập.
- Thứ tự vào phòng chờ phải công bằng (FIFO theo thời điểm request tới) trừ khi có chính sách ưu tiên rõ ràng được công bố trước (ví dụ khách VIP), tránh cảm giác không công bằng cho user thường.
- Hệ thống throttle phải điều chỉnh động số lượng cho qua dựa trên tình trạng thực tế của backend (ví dụ giảm số cho qua nếu latency DB tăng cao), không dùng số cố định tĩnh không phản ánh tải thật.
- Đảm bảo user đã vào được luồng xử lý (qua khỏi hàng chờ) không bị rate limit lần nữa giữa đường gây trải nghiệm xấu (đã chờ xong lại bị chặn tiếp).
- Đo lường và báo cáo: throughput checkout thành công/giây tối đa hệ thống đạt được an toàn, và tỉ lệ user phải vào hàng chờ so với tổng traffic trong giờ flash sale, dùng để lên kế hoạch capacity cho các sự kiện sau.

---

# Base ERD — Checkout gọi thẳng backend, chưa có phòng chờ

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có throttle ở tầng vào (virtual waiting room). Đề bài nói e-commerce có flash sale với sản phẩm giới hạn số lượng, user checkout trực tiếp — nên base cần đủ: user, sản phẩm flash sale, và order được tạo ngay khi checkout, không có khái niệm hàng chờ/priority/admission token/đo throughput. Những entity đó là phần enhance.

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    PRODUCT ||--o{ ORDER : "ordered as"

    USER {
        string id PK
        string name
    }
    PRODUCT {
        string id PK
        string name
        int flash_sale_stock
        decimal price
    }
    ORDER {
        string id PK
        string user_id FK
        string product_id FK
        string status "pending | paid | failed"
        datetime created_at
    }
```

---

# Base sequence — Checkout (gọi thẳng backend, không có admission control)

Đây là **base**, flow "Checkout" ở trạng thái hiện tại — request đi thẳng từ client vào backend (DB, payment) không qua bất kỳ điểm kiểm soát lượng vào nào. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (backend quá tải, không công bằng khi nghẽn, không đo lường được) đều bắt nguồn từ việc thiếu tầng throttle ở đây.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    actor UserN as ... hàng nghìn user khác
    participant Checkout as Checkout Service
    participant DB as Order/Inventory DB
    participant Payment as Payment Service

    Note over UserA,UserN: Giờ flash sale, tất cả cùng bấm "Mua ngay" trong vài giây

    par Toàn bộ request dồn thẳng vào backend cùng lúc
        UserA->>Checkout: POST /checkout
        Checkout->>DB: Kiểm tra + trừ flash_sale_stock, tạo ORDER
    and
        UserB->>Checkout: POST /checkout
        Checkout->>DB: Kiểm tra + trừ flash_sale_stock, tạo ORDER
    and
        UserN->>Checkout: POST /checkout
        Checkout->>DB: Kiểm tra + trừ flash_sale_stock, tạo ORDER
    end

    DB-->>Checkout: Latency tăng cao do quá nhiều connection đồng thời
    Note over DB: Không có cơ chế nào giới hạn số request được vào cùng lúc, DB có thể timeout hoặc sập hoàn toàn

    Checkout->>Payment: Gọi thanh toán (cho các request DB xử lý kịp)
    Checkout-->>UserA: Timeout / lỗi 500
    Checkout-->>UserB: Timeout / lỗi 500

    Note over Checkout,UserN: Không ai biết mình đang "xếp hàng" hay đã thất bại hẳn, không có ước tính thời gian chờ
```

---

# Enhance ERD — sau khi có virtual waiting room + admission control

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `USER` bổ sung `tier` — cơ sở cho chính sách ưu tiên công bố trước (VIP) khi hàng chờ đông.
- `WAITING_ROOM_QUEUE_ENTRY` (mới) — vị trí, thời điểm vào hàng, trạng thái chờ/đã vào/hết hạn.
- `ADMISSION_TOKEN` (mới) — cấp cho user đã qua hàng chờ, dùng để checkout mà không bị throttle lại giữa đường.
- `ADMISSION_RATE_CONFIG` + `BACKEND_HEALTH_SNAPSHOT` (mới) — tốc độ cho qua hiện tại được điều chỉnh động theo sức khỏe backend thực tế, không dùng số tĩnh.
- `THROUGHPUT_METRIC` (mới) — throughput checkout thành công/giây và tỉ lệ user phải chờ, phục vụ lên kế hoạch capacity.

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    PRODUCT ||--o{ ORDER : "ordered as"
    USER ||--o{ WAITING_ROOM_QUEUE_ENTRY : joins
    WAITING_ROOM_QUEUE_ENTRY ||--o| ADMISSION_TOKEN : grants
    ADMISSION_TOKEN ||--o| ORDER : authorizes
    BACKEND_HEALTH_SNAPSHOT ||--o{ ADMISSION_RATE_CONFIG : informs

    USER {
        string id PK
        string name
        string tier "regular | vip"
    }
    PRODUCT {
        string id PK
        string name
        int flash_sale_stock
        decimal price
    }
    ORDER {
        string id PK
        string user_id FK
        string product_id FK
        string status "pending | paid | failed"
        datetime created_at
    }
    WAITING_ROOM_QUEUE_ENTRY {
        string id PK
        string user_id FK
        string priority_tier "vip | regular"
        int position
        datetime joined_at
        string status "waiting | admitted | expired"
    }
    ADMISSION_TOKEN {
        string id PK
        string queue_entry_id FK
        string user_id FK
        datetime issued_at
        datetime expires_at
        boolean used_for_checkout
    }
    ADMISSION_RATE_CONFIG {
        string id PK
        int current_admit_rate_per_sec
        string reason "healthy | db_latency_high | error_rate_high"
        datetime updated_at
    }
    BACKEND_HEALTH_SNAPSHOT {
        string id PK
        datetime measured_at
        int db_latency_ms
        decimal error_rate_pct
        string status "healthy | degraded"
    }
    THROUGHPUT_METRIC {
        string id PK
        datetime window_start
        datetime window_end
        decimal successful_checkouts_per_sec
        int total_user_count
        int queued_user_count
        decimal queued_ratio_pct
    }
```

---

# Enhance sequence — Checkout (qua phòng chờ, admission token miễn throttle lại)

Đây là **enhance** của flow "Checkout" đã có ở base. So với base, request không còn đi thẳng vào backend — user bị chặn ở virtual waiting room theo thứ tự FIFO (trừ khi có priority tier công bố trước), chỉ khi tới lượt mới nhận admission token; sau khi cầm token, user checkout thẳng qua backend mà không bị rate limit lần nữa (đáp ứng yêu cầu 1, 2 và 4 của đề bài).

```mermaid
sequenceDiagram
    actor VIP as User VIP
    actor Reg as User thường
    participant WR as Waiting Room Service
    participant Queue as WAITING_ROOM_QUEUE_ENTRY
    participant Token as ADMISSION_TOKEN
    participant Checkout as Checkout Service
    participant DB as Order/Inventory DB

    par Cao điểm, nhiều user cùng bấm mua
        Reg->>WR: POST /checkout
        WR->>Queue: Thêm QUEUE_ENTRY(priority_tier=regular, joined_at=now)
    and
        VIP->>WR: POST /checkout
        WR->>Queue: Thêm QUEUE_ENTRY(priority_tier=vip, joined_at=now)
    end

    Note over Queue: Thứ tự xử lý FIFO theo joined_at trong cùng priority_tier, VIP được công bố trước là ưu tiên hơn regular

    WR-->>Reg: Hiển thị vị trí hàng chờ và thời gian ước tính
    WR-->>VIP: Hiển thị vị trí hàng chờ (ngắn hơn nhờ priority)

    loop Tới lượt theo admit rate hiện tại
        Queue->>Token: Cấp ADMISSION_TOKEN cho entry đầu hàng
        Queue->>Queue: Đánh dấu QUEUE_ENTRY(status=admitted)
    end

    Token-->>VIP: Nhận token, chuyển vào luồng checkout
    VIP->>Checkout: POST /checkout (kèm admission_token)
    Checkout->>Token: Xác thực token còn hạn, chưa dùng
    Note over Checkout,Token: Token hợp lệ nên bỏ qua mọi rate limit khác, không bắt user chờ thêm lần nữa
    Checkout->>DB: Trừ flash_sale_stock, tạo ORDER
    DB-->>Checkout: Thành công
    Checkout-->>VIP: Checkout thành công
    Checkout->>Token: Đánh dấu used_for_checkout=true
```

---

# Enhance sequence — Điều chỉnh admission rate động theo sức khỏe backend

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm admit rate). Đáp ứng yêu cầu 3 của đề bài: tốc độ cho qua hàng chờ phải phản ánh tải thật của backend, không dùng số cố định tĩnh.

```mermaid
sequenceDiagram
    participant DB as Order/Inventory DB
    participant Health as Backend Health Monitor
    participant Snapshot as BACKEND_HEALTH_SNAPSHOT
    participant Config as ADMISSION_RATE_CONFIG
    participant WR as Waiting Room Service

    loop Định kỳ (ví dụ mỗi vài giây)
        Health->>DB: Đo latency, error rate hiện tại
        Health->>Snapshot: Ghi BACKEND_HEALTH_SNAPSHOT(db_latency_ms, error_rate_pct)

        alt Latency/error rate vượt ngưỡng an toàn
            Snapshot->>Config: Yêu cầu giảm current_admit_rate_per_sec
            Config->>Config: Cập nhật rate mới, reason=db_latency_high
        else Backend đang khỏe, còn dư sức chịu tải
            Snapshot->>Config: Yêu cầu tăng dần current_admit_rate_per_sec (không tăng đột ngột)
            Config->>Config: Cập nhật rate mới, reason=healthy
        end
    end

    WR->>Config: Đọc current_admit_rate_per_sec trước mỗi vòng cấp ADMISSION_TOKEN

    Note over WR,Config: Số lượng user được cấp token mỗi vòng luôn theo rate mới nhất, phản ánh tải thật thay vì con số cấu hình cứng
```

---

# Enhance sequence — Đo throughput và tỉ lệ user phải chờ để lên kế hoạch capacity

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không đo lường throughput hay tỉ lệ user bị chặn). Đáp ứng yêu cầu 5 của đề bài: đo throughput checkout thành công/giây tối đa an toàn, và tỉ lệ user phải vào hàng chờ so với tổng traffic, phục vụ capacity planning cho các sự kiện sau.

```mermaid
sequenceDiagram
    participant Checkout as Checkout Service
    participant WR as Waiting Room Service
    participant Reporter as Metrics Reporter (job định kỳ)
    participant Metric as THROUGHPUT_METRIC
    actor Planner as Đội capacity planning

    loop Suốt giờ flash sale
        Checkout->>Reporter: Ghi nhận mỗi checkout thành công
        WR->>Reporter: Ghi nhận mỗi user vào hàng chờ / được admit
    end

    loop Cuối mỗi window (ví dụ mỗi phút)
        Reporter->>Reporter: Tính successful_checkouts_per_sec trong window
        Reporter->>Reporter: Tính queued_ratio_pct = queued_user_count / total_user_count
        Reporter->>Metric: Ghi THROUGHPUT_METRIC(window_start, window_end, successful_checkouts_per_sec, queued_ratio_pct)
    end

    Planner->>Metric: Truy vấn throughput tối đa an toàn đã đạt được trong sự kiện
    Metric-->>Planner: Trả về throughput đỉnh và tỉ lệ user từng phải chờ

    Note over Planner,Metric: Dữ liệu này dùng để ước tính backend cần scale thêm bao nhiêu, hoặc admit rate nên đặt bao nhiêu cho flash sale kế tiếp
```
