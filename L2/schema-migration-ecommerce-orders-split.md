# Tách bảng `orders` thành `orders` + `order_items` không downtime (e-commerce)

**Hệ thống:** Một sàn e-commerce có bảng `orders` cũ lưu items dưới dạng JSON trong 1 cột, cần tách sang bảng quan hệ `order_items` để hỗ trợ báo cáo/join hiệu quả hơn.

**Vai trò của flow:** Trong lúc traffic đặt hàng vẫn chạy 24/7, flow phải đảm bảo mọi order mới tạo được ghi cả vào cột JSON cũ và bảng `order_items` mới, đồng thời có job backfill order cũ, mà không làm gián đoạn checkout.

**Yêu cầu cụ thể:**
- Yêu cầu transaction tạo order dual-write phải wrap trong 1 transaction DB duy nhất (ghi `orders` + insert `order_items`) để không có trường hợp order được tạo nhưng thiếu order_items do crash giữa chừng.
- Mô tả cụ thể tình huống 2 request checkout đồng thời cho cùng 1 cart (double-submit do user bấm 2 lần) trong giai đoạn dual-write: yêu cầu idempotency key theo `cart_id` + `checkout_attempt_id` để chỉ 1 trong 2 request tạo order thành công, request kia phải nhận lại kết quả của request đầu (không tạo order trùng ở cả bảng cũ và mới).
- Job backfill order lịch sử phải chạy bằng cursor theo `order_id` tăng dần theo batch nhỏ (ví dụ 500 order/lần), có checkpoint để resume nếu job bị crash giữa chừng, không đọc lại từ đầu.
- Định nghĩa cơ chế kiểm tra tính nhất quán (reconciliation job) so sánh số lượng/tổng tiền giữa cột JSON cũ và bảng `order_items` mới cho một mẫu order ngẫu nhiên mỗi ngày, cảnh báo nếu phát hiện lệch.
- Quy định rõ điều kiện để an toàn drop cột JSON cũ: phải chạy dual-write ổn định (không lệch dữ liệu) trong một khoảng thời gian tối thiểu, và đã chuyển 100% luồng đọc sang bảng mới.

---

# ERD — Base (trước khi tách order_items)

Đây là **base**: mô hình dữ liệu suy luận cho sàn e-commerce *trước khi* tách bảng. Đề bài giả định đã tồn tại bảng `orders` lưu danh sách sản phẩm của đơn hàng dưới dạng JSON trong 1 cột — nếu không có sẵn `orders` với cột JSON đó thì yêu cầu "tách sang bảng order_items" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ tạo đơn hàng, không suy diễn thêm các module không liên quan (khuyến mãi, vận chuyển, thanh toán...).

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER }o--o{ PRODUCT : "items embedded in items_json, no real FK"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        string items_json
        decimal total_amount
        string status
        datetime created_at
    }

    PRODUCT {
        string product_id PK
        string name
        decimal price
        int stock_quantity
    }
```

---

<a id="2-base-checkout-create-order-sqd"></a>

# Sequence Diagram — Base: Checkout Create Order

Đây là **base**, flow "tạo đơn hàng lúc checkout" — tiền đề bắt buộc cho việc tách bảng: chính flow này đang ghi toàn bộ danh sách item vào cột `items_json` của bảng `orders`, là nguồn dữ liệu duy nhất cần được dual-write sang bảng `order_items` mới ở enhance.

```mermaid
sequenceDiagram
    actor Customer
    participant Checkout as Checkout Service
    participant OrderSvc as Order Service
    participant DB as Orders Table

    Customer->>Checkout: Submit checkout for cart
    Checkout->>Checkout: Validate cart items, compute total
    Checkout->>OrderSvc: Create order (cart items, total)
    OrderSvc->>DB: INSERT INTO orders (items_json, total_amount, status)
    DB-->>OrderSvc: order_id
    OrderSvc-->>Checkout: Order created
    Checkout-->>Customer: Checkout success, order confirmation
```

---

# ERD — Enhance (sau khi tách order_items)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu của đề bài — dual-write `order_items` quan hệ song song với `items_json` cũ, checkpoint cho job backfill, và báo cáo đối soát. So với base, `ORDER` giữ nguyên cột `items_json` (chưa drop, vẫn đang dual-write) và có thêm `idempotency_key` để chống double-submit; quan hệ `ORDER` — `PRODUCT` giờ đi qua `ORDER_ITEM` thật sự là FK thay vì chỉ nằm lỏng lẻo trong JSON.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : "contains (new, dual-written)"
    ORDER_ITEM }o--|| PRODUCT : references
    RECONCILIATION_REPORT }o--|| ORDER : samples

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        string items_json
        string idempotency_key
        decimal total_amount
        string status
        datetime created_at
    }

    ORDER_ITEM {
        string order_item_id PK
        string order_id FK
        string product_id FK
        int quantity
        decimal unit_price
    }

    PRODUCT {
        string product_id PK
        string name
        decimal price
        int stock_quantity
    }

    BACKFILL_CHECKPOINT {
        string checkpoint_id PK
        string last_order_id_processed
        int batch_size
        string status
        datetime updated_at
    }

    RECONCILIATION_REPORT {
        string report_id PK
        date run_date
        string sample_order_id FK
        decimal json_total
        decimal relational_total
        boolean mismatch_found
    }
```

---

# Sequence Diagram — Enhance: Checkout Create Order

Đây là **enhance**, flow "tạo đơn hàng lúc checkout" đã tồn tại ở base ([2-base-checkout-create-order-sqd.md](#2-base-checkout-create-order-sqd)) nay thay đổi lớn: ghi `orders` (items_json) và `order_items` phải nằm trong cùng 1 transaction DB, và phải chống double-submit bằng idempotency key theo `cart_id` + `checkout_attempt_id` khi user bấm checkout 2 lần gần như đồng thời.

```mermaid
sequenceDiagram
    actor Customer
    participant Checkout as Checkout Service
    participant OrderSvc as Order Service
    participant DB as Orders + Order_Items Tables

    par Double-submit, 2 requests gần như đồng thời
        Customer->>Checkout: Submit checkout (request 1, cart_id, attempt_id)
    and
        Customer->>Checkout: Submit checkout (request 2, cart_id, same attempt_id)
    end

    Checkout->>OrderSvc: Create order, idempotency_key = cart_id + attempt_id

    alt Request đầu tiên tới trước, chưa có idempotency_key
        OrderSvc->>DB: BEGIN TRANSACTION
        OrderSvc->>DB: INSERT INTO orders (items_json, idempotency_key, total_amount)
        OrderSvc->>DB: INSERT INTO order_items (order_id, product_id, quantity, unit_price) for each item
        OrderSvc->>DB: COMMIT
        DB-->>OrderSvc: order_id created
        OrderSvc-->>Checkout: Order created
    else Request thứ hai, idempotency_key đã tồn tại
        OrderSvc->>DB: Look up existing order by idempotency_key
        DB-->>OrderSvc: Existing order_id found
        OrderSvc-->>Checkout: Return existing order result, no new order created
    end

    Checkout-->>Customer: Checkout success, order confirmation
```

---

# Sequence Diagram — Enhance: Backfill Historical Orders

Đây là **enhance**, flow hoàn toàn mới: job backfill đọc các order lịch sử (tạo trước khi có dual-write) theo cursor `order_id` tăng dần, batch nhỏ 500 order/lần, có checkpoint để resume nếu crash giữa chừng thay vì đọc lại từ đầu.

```mermaid
sequenceDiagram
    participant Job as Backfill Job
    participant Checkpoint as Backfill Checkpoint Store
    participant DB as Orders + Order_Items Tables

    Job->>Checkpoint: Read last_order_id_processed
    Checkpoint-->>Job: Resume cursor (or start from 0 if none)

    loop Until no more orders
        Job->>DB: SELECT next 500 orders WHERE order_id > cursor ORDER BY order_id
        DB-->>Job: Batch of orders with items_json

        loop for each order in batch
            Job->>Job: Parse items_json into item rows
            Job->>DB: INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        end

        Job->>Checkpoint: Update last_order_id_processed = max order_id in batch
        Checkpoint-->>Job: Checkpoint saved
    end

    Note over Job,Checkpoint: Nếu job crash giữa batch, lần chạy sau resume đúng từ checkpoint đã lưu, không đọc lại từ đầu
    Job-->>Job: Backfill completed
```

---

# Sequence Diagram — Enhance: Reconciliation Check

Đây là **enhance**, flow hoàn toàn mới: job đối soát hằng ngày so sánh số lượng/tổng tiền giữa cột `items_json` cũ và bảng `order_items` mới cho một mẫu order ngẫu nhiên, cảnh báo nếu lệch — điều kiện bắt buộc trước khi được phép drop cột JSON cũ.

```mermaid
sequenceDiagram
    participant Job as Reconciliation Job
    participant DB as Orders + Order_Items Tables
    participant Alert as Alert Service

    Job->>DB: Pick random sample of orders for today
    DB-->>Job: Sample order list

    loop for each sampled order
        Job->>DB: Read items_json for order
        Job->>DB: Read order_items rows for order
        Job->>Job: Compute total/count from items_json
        Job->>Job: Compute total/count from order_items
        alt Totals match
            Job->>Job: Record report, mismatch_found = false
        else Totals differ
            Job->>Job: Record report, mismatch_found = true
            Job->>Alert: Trigger alert (order_id, json_total, relational_total)
        end
    end

    Job-->>Job: Reconciliation run completed
    Note over Job: Chỉ khi dual-write chạy ổn định, không lệch trong thời gian tối thiểu quy định và 100% đọc đã chuyển sang order_items thì mới được drop items_json
```
