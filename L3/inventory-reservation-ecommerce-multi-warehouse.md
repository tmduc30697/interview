# Giữ tồn kho khi thêm vào giỏ hàng trên sàn e-commerce nhiều kho

**Hệ thống:** Một sàn e-commerce có tồn kho phân bổ ở nhiều kho khác nhau theo khu vực, khách hàng thêm sản phẩm vào giỏ và tồn kho được "soft reserve" trong 15 phút khi vào bước checkout.

**Vai trò của flow:** Flow reservation phải chọn đúng kho gần khách để giữ hàng, trừ tồn kho tạm thời (không phải trừ thật) khi vào checkout, và giải phóng đúng nếu khách rời trang mà không thanh toán.

**Yêu cầu cụ thể:**
- Khi 2 khách ở 2 khu vực khác nhau cùng checkout sản phẩm chỉ còn 1 đơn vị tồn kho ở kho gần cả hai, dùng update nguyên tử `UPDATE inventory SET reserved = reserved + 1 WHERE available - reserved > 0` (không phải đọc số dư trước rồi update sau) để chỉ 1 trong 2 giữ được hàng, người thua nhận thông báo ngay và được đề xuất kho khác (nếu có) hoặc thông báo hết hàng.
- Reservation phải có TTL và bảng lưu trạng thái riêng (không chỉ tăng/giảm số nguyên trong cột `reserved`), để có thể truy ngược "ai đang giữ, giữ bao lâu, khi nào hết hạn" phục vụ debug và hiển thị đúng cho khách.
- Mô tả cụ thể race giữa job giải phóng reservation hết hạn và request thanh toán đến gần đồng thời: transaction thanh toán phải kiểm tra reservation vẫn còn hiệu lực (chưa bị job expire xóa) ngay trước khi trừ tồn kho thật, nếu đã bị expire phải từ chối thanh toán rõ ràng và không charge tiền.
- Khi khách sửa số lượng trong giỏ hàng (tăng từ 1 lên 3) trong lúc đang giữ reservation, phải tính lại đúng phần chênh lệch cần giữ thêm một cách atomic, không phải hủy reservation cũ rồi tạo mới (dễ mất tồn kho đã giữ trong khoảng thời gian giữa 2 bước).
- Có cơ chế đồng bộ giữa tồn kho "hiển thị công khai trên trang sản phẩm" và tồn kho "khả dụng thực" (available - reserved), quy định độ trễ chấp nhận được và tránh hiển thị "còn hàng" khi thực chất đã được reserve hết bởi các giỏ hàng khác.

---

# ERD - Base (trước khi có reservation atomic nhiều kho)

Đây là **base**: mô hình dữ liệu của sàn e-commerce nhiều kho, nơi tồn kho đã có khái niệm "soft reserve 15 phút" nhưng được cài đặt đơn giản - `reserved` chỉ là một cột số nguyên tăng/giảm trực tiếp trên `Inventory`, không có bảng reservation riêng để truy ngược ai đang giữ, giữ từ khi nào, hết hạn lúc nào. `CartItem` chỉ lưu `reserved_at` để một cron job dựa vào đó blind-decrement `reserved` sau 15 phút.

```mermaid
erDiagram
    CUSTOMER ||--o{ CART : owns
    CUSTOMER ||--o{ CART_ITEM : reserves_via
    CART ||--o{ CART_ITEM : contains
    WAREHOUSE ||--o{ INVENTORY : stocks
    PRODUCT ||--o{ INVENTORY : tracked_in
    INVENTORY ||--o{ CART_ITEM : reserved_from

    CUSTOMER {
        string id PK
        string region
    }

    WAREHOUSE {
        string id PK
        string name
        string region
    }

    PRODUCT {
        string id PK
        string title
    }

    INVENTORY {
        string id PK
        string product_id FK
        string warehouse_id FK
        int available
        int reserved
    }

    CART {
        string id PK
        string customer_id FK
    }

    CART_ITEM {
        string id PK
        string cart_id FK
        string product_id FK
        string warehouse_id FK
        int quantity
        datetime reserved_at
    }
```

---

# Sequence - Base - Flow "checkout-reserve"

Đây là **base**: khách vào bước checkout, hệ thống chọn kho gần nhất rồi giữ tồn kho bằng cách đọc số dư trước, kiểm tra, rồi ghi sau (2 bước tách rời, không atomic). Flow này được chọn vì nó chính là nguyên nhân của race mà yêu cầu 1 mô tả: 2 khách ở 2 khu vực khác nhau cùng checkout sản phẩm chỉ còn 1 đơn vị ở kho gần cả hai.

```mermaid
sequenceDiagram
    actor CustomerA as Khách A (khu vực gần kho X)
    actor CustomerB as Khách B (khu vực gần kho X)
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: available = 1, reserved = 0 tại kho X

    CustomerA->>App: vào checkout sản phẩm Y
    App->>App: chọn kho gần nhất, tìm được kho X
    App->>Inventory: đọc available - reserved (đọc = 1)

    CustomerB->>App: vào checkout sản phẩm Y gần như cùng lúc
    App->>App: chọn kho gần nhất, cũng là kho X
    App->>Inventory: đọc available - reserved (đọc = 1, chưa thấy thay đổi của A)

    App-->>App: (Khách A) 1 > 0, hợp lệ
    App->>Inventory: ghi reserved = reserved + 1

    App-->>App: (Khách B) dựa trên giá trị đã đọc trước đó, cũng hợp lệ
    App->>Inventory: ghi reserved = reserved + 1

    Note over Inventory: reserved = 2 nhưng available chỉ có 1,\ncả 2 khách đều tưởng giữ được hàng
```

---

# Sequence - Base - Flow "update-cart-quantity"

Đây là **base**: khi khách sửa số lượng trong giỏ hàng lúc đang giữ reservation, hệ thống hủy phần đã giữ rồi giữ lại từ đầu theo số lượng mới - 2 bước tách rời. Flow này được chọn vì đây chính là cách làm mà yêu cầu 4 chỉ ra là có vấn đề: giữa lúc hủy và giữ lại, tồn kho đã giữ có thể bị người khác giành mất.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: reserved = 1 (do khách này giữ số lượng 1 trước đó)

    Customer->>App: sửa số lượng trong giỏ từ 1 lên 3
    App->>Inventory: bước 1, giải phóng reserved -= 1 (reserved về 0)

    Note over Inventory: Khoảng hở giữa 2 bước, người khác có thể giành tồn kho ngay lúc này

    App->>Inventory: bước 2, giữ lại reserved += 3 cho số lượng mới

    alt Không ai giành tồn kho trong khoảng hở
        Inventory-->>App: giữ thành công 3 đơn vị
        App-->>Customer: cập nhật giỏ hàng thành công
    else Khách khác giành mất tồn kho trong khoảng hở
        Inventory-->>App: không đủ tồn kho để giữ 3 đơn vị
        App-->>Customer: báo lỗi, tồn kho đã bị giành dù khách này đã giữ trước đó
    end
```

---

# Sequence - Base - Flow "expire-reservation"

Đây là **base**: cron job giải phóng reservation hết hạn bằng cách blind-decrement cột `reserved`, dựa trên `reserved_at` của `CartItem`, không kiểm tra gì thêm. Flow này được chọn vì nó là tiền đề cho yêu cầu 3 - khi job này chạy gần như đồng thời với request thanh toán, không có cơ chế nào đảm bảo thứ tự đúng, có thể vừa charge tiền vừa đã giải phóng tồn kho cho người khác.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)
    participant ExpireJob as Cron Expire Job

    Note over Inventory: CartItem.reserved_at cách đây 15 phút, reserved = 1

    par Request thanh toán tới gần như đồng thời
        Customer->>App: xác nhận thanh toán
        App->>Inventory: trừ tồn kho thật, available -= 1
        App-->>Customer: thanh toán thành công, charge tiền
    and Cron job chạy đúng lúc đó
        ExpireJob-->>ExpireJob: quét CartItem có reserved_at quá 15 phút
        ExpireJob->>Inventory: reserved -= 1 (giải phóng, không kiểm tra thanh toán có đang chạy không)
    end

    Note over Inventory: Không rõ thứ tự nào xảy ra trước,\ncó thể vừa charge tiền khách vừa giải phóng tồn kho cho người khác giữ tiếp
```

---

# ERD - Enhance (reservation atomic + bảng trạng thái riêng)

Đây là **enhance**: mô hình dữ liệu sau khi tách reservation ra khỏi cột `reserved` đơn thuần. So với base, các thay đổi là:

- `RESERVATION` (mới, thay cho việc chỉ tăng/giảm `Inventory.reserved`): mỗi lần giữ hàng là 1 dòng riêng với `status` (`active` / `expired` / `consumed`), `created_at`, `expires_at` - truy ngược được ai đang giữ, giữ bao lâu, hết hạn lúc nào (yêu cầu 2), và cho phép transaction thanh toán kiểm tra `status = active` ngay trước khi trừ tồn kho thật (yêu cầu 3).
- `INVENTORY` có thêm `version` để hỗ trợ update nguyên tử `available - reserved > 0` (yêu cầu 1) và điều chỉnh phần chênh lệch khi khách sửa số lượng mà không cần hủy/tạo lại (yêu cầu 4).
- `CART_ITEM` giờ tham chiếu tới `RESERVATION` qua `reservation_id` thay vì tự lưu `reserved_at`.
- `PRODUCT_STOCK_DISPLAY` (mới): snapshot tồn kho hiển thị công khai trên trang sản phẩm, đồng bộ định kỳ từ `available - reserved` thật, có `sync_lag_seconds` để biết độ trễ hiện tại - tránh hiển thị "còn hàng" khi thực chất đã được reserve hết (yêu cầu 5).

```mermaid
erDiagram
    CUSTOMER ||--o{ CART : owns
    CUSTOMER ||--o{ RESERVATION : holds
    CART ||--o{ CART_ITEM : contains
    WAREHOUSE ||--o{ INVENTORY : stocks
    PRODUCT ||--o{ INVENTORY : tracked_in
    INVENTORY ||--o{ RESERVATION : reserved_from
    CART_ITEM ||--o| RESERVATION : backed_by
    PRODUCT ||--o{ PRODUCT_STOCK_DISPLAY : displayed_as

    CUSTOMER {
        string id PK
        string region
    }

    WAREHOUSE {
        string id PK
        string name
        string region
    }

    PRODUCT {
        string id PK
        string title
    }

    INVENTORY {
        string id PK
        string product_id FK
        string warehouse_id FK
        int available
        int reserved
        int version
    }

    CART {
        string id PK
        string customer_id FK
    }

    CART_ITEM {
        string id PK
        string cart_id FK
        string product_id FK
        string warehouse_id FK
        int quantity
        string reservation_id FK
    }

    RESERVATION {
        string id PK
        string inventory_id FK
        string customer_id FK
        int quantity
        string status
        datetime created_at
        datetime expires_at
    }

    PRODUCT_STOCK_DISPLAY {
        string id PK
        string product_id FK
        int displayed_available
        datetime synced_at
        int sync_lag_seconds
    }
```

---

# Sequence - Enhance - Flow "checkout-reserve"

Đây là **enhance**, cùng flow `checkout-reserve` như ở base nhưng thay đọc-rồi-ghi bằng update nguyên tử `UPDATE inventory SET reserved = reserved + 1 WHERE available - reserved > 0`, đáp ứng yêu cầu 1: chỉ 1 trong 2 khách giữ được hàng, người thua nhận thông báo ngay và được đề xuất kho khác hoặc báo hết hàng. Mỗi lần giữ thành công đều tạo 1 dòng `Reservation` có TTL riêng, đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    actor CustomerA as Khách A (khu vực gần kho X)
    actor CustomerB as Khách B (khu vực gần kho X)
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: available = 1, reserved = 0 tại kho X

    par Khách A vào checkout
        CustomerA->>App: vào checkout sản phẩm Y
        App->>Inventory: UPDATE reserved = reserved + 1 WHERE available - reserved > 0
        Inventory-->>App: 1 dòng bị ảnh hưởng, thành công
        App->>App: tạo Reservation, status = active, expires_at = now + 15 phút
        App-->>CustomerA: giữ hàng thành công tại kho X
    and Khách B vào checkout gần như cùng lúc
        CustomerB->>App: vào checkout sản phẩm Y
        App->>Inventory: UPDATE reserved = reserved + 1 WHERE available - reserved > 0
        Inventory-->>App: 0 dòng bị ảnh hưởng, điều kiện không còn đúng
        App-->>App: tìm kho khác còn hàng gần khách B
        alt Có kho khác còn hàng
            App-->>CustomerB: đề xuất giữ hàng ở kho khác (thời gian giao có thể lâu hơn)
        else Không còn kho nào khác
            App-->>CustomerB: thông báo hết hàng ngay lập tức
        end
    end
```

---

# Sequence - Enhance - Flow "expire-reservation"

Đây là **enhance**, cùng flow `expire-reservation` như ở base nhưng thay blind-decrement bằng kiểm tra trạng thái tường minh trên bảng `Reservation`, đáp ứng yêu cầu 3: transaction thanh toán phải kiểm tra reservation vẫn còn `status = active` ngay trước khi trừ tồn kho thật, nếu đã bị job expire chuyển sang `expired` thì từ chối thanh toán rõ ràng và không charge tiền - thay vì tình trạng mơ hồ như ở base.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)
    participant ExpireJob as Cron Expire Job

    Note over Inventory: Reservation.status = active, expires_at đã qua 15 phút

    par Request thanh toán tới gần như đồng thời
        Customer->>App: xác nhận thanh toán
        App->>Inventory: BEGIN transaction, khóa dòng Reservation (SELECT FOR UPDATE)
        App-->>App: kiểm tra Reservation.status = active và chưa bị expire job xử lý
        alt Reservation vẫn active tại thời điểm khóa
            App->>Inventory: trừ tồn kho thật, available -= 1, reserved -= 1
            App->>App: Reservation.status = consumed
            App-->>Customer: thanh toán thành công, charge tiền
        else Reservation đã bị expire job chuyển sang expired trước khi khóa được
            App-->>Customer: từ chối thanh toán rõ ràng, reservation đã hết hạn, không charge tiền
        end
    and Cron job chạy đúng lúc đó
        ExpireJob-->>ExpireJob: quét Reservation có expires_at đã qua và status = active
        ExpireJob->>Inventory: UPDATE reservation SET status = expired WHERE status = active
        ExpireJob->>Inventory: reserved -= 1 (chỉ khi update status ở trên thành công)
    end

    Note over App,ExpireJob: Chỉ 1 trong 2 nhánh thắng nhờ khóa dòng Reservation,\nkhông bao giờ vừa charge tiền vừa đã giải phóng tồn kho
```

---

# Sequence - Enhance - Flow "update-cart-quantity"

Đây là **enhance**, cùng flow `update-cart-quantity` như ở base nhưng thay hủy-rồi-giữ-lại bằng 1 update nguyên tử tính đúng phần chênh lệch, đáp ứng yêu cầu 4: khi khách sửa số lượng từ 1 lên 3, hệ thống chỉ cần giữ thêm phần chênh lệch (2 đơn vị) trong cùng 1 thao tác, không có khoảng hở nào để người khác giành mất tồn kho đã giữ.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: Reservation hiện tại của khách này, quantity = 1, status = active

    Customer->>App: sửa số lượng trong giỏ từ 1 lên 3
    App-->>App: tính delta = 3 - 1 = 2 (phần cần giữ thêm)

    App->>Inventory: UPDATE reserved = reserved + 2 WHERE available - reserved >= 2
    alt Đủ tồn kho cho phần chênh lệch
        Inventory-->>App: thành công
        App->>App: cập nhật Reservation.quantity = 3 trong cùng transaction
        App-->>Customer: cập nhật giỏ hàng thành công, vẫn giữ nguyên phần đã có
    else Không đủ tồn kho cho phần chênh lệch
        Inventory-->>App: thất bại, không đủ 2 đơn vị còn lại
        App-->>Customer: báo không thể tăng số lượng, giữ nguyên reservation cũ (vẫn còn 1)
    end
```

---

# Sequence - Enhance - Flow "stock-display-sync"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 5: đồng bộ giữa tồn kho hiển thị công khai trên trang sản phẩm và tồn kho khả dụng thực (`available - reserved`), quy định độ trễ chấp nhận được và tránh hiển thị "còn hàng" khi thực chất đã được reserve hết bởi các giỏ hàng khác.

```mermaid
sequenceDiagram
    actor Shopper as Khách đang xem trang sản phẩm
    participant App as E-commerce App
    participant Inventory as Inventory (tổng nhiều kho)
    participant Display as Product Stock Display
    participant SyncJob as Stock Display Sync Job

    loop Mỗi vài giây (độ trễ chấp nhận được, ví dụ tối đa 5 giây)
        SyncJob->>Inventory: tính available - reserved theo từng sản phẩm, gộp tất cả kho
        Inventory-->>SyncJob: available_thuc_te
        SyncJob->>Display: cập nhật displayed_available = available_thuc_te,\nsynced_at = now, sync_lag_seconds = độ trễ đo được
    end

    Shopper->>App: xem trang sản phẩm Y
    App->>Display: đọc displayed_available (snapshot đã đồng bộ gần nhất)
    Display-->>App: displayed_available = 0 (đã bị reserve hết bởi giỏ hàng khác)
    App-->>Shopper: hiển thị "Hết hàng" thay vì "Còn hàng" dựa trên số cũ đã lỗi thời

    Note over Display: Nếu sync_lag_seconds vượt ngưỡng chấp nhận được,\nhệ thống cảnh báo để tránh hiển thị sai kéo dài
```
