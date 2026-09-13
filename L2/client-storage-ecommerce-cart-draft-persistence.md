# Giữ giỏ hàng và form checkout dở dang qua localStorage

**Hệ thống:** SPA e-commerce lưu giỏ hàng và form checkout dở dang vào localStorage để user không mất dữ liệu khi reload/đóng nhầm tab, đồng thời phải đồng bộ đúng với giỏ hàng thật trên server khi đăng nhập.

**Vai trò của flow:** Đảm bảo trải nghiệm "không mất dữ liệu" khi mất mạng/đóng nhầm tab, nhưng không để dữ liệu cache phía client lấn át dữ liệu nguồn (server) gây sai lệch giá/tồn kho.

**Yêu cầu cụ thể:**
- Khi user thêm sản phẩm vào giỏ ở trạng thái chưa đăng nhập (guest), giỏ hàng lưu localStorage; khi đăng nhập, phải merge đúng với giỏ hàng đã có sẵn trên server của tài khoản đó — quyết định rõ thứ tự ưu tiên (giữ số lượng lớn hơn? cộng dồn? hỏi user khi conflict?) chứ không được ghi đè ngầm định gây mất dữ liệu.
- localStorage lưu snapshot giá tại thời điểm thêm vào giỏ; trước khi checkout phải revalidate lại giá/tồn kho thật với server, và xử lý rõ khi giá đổi hoặc hết hàng (hiển thị cảnh báo, không để user thanh toán nhầm giá cũ).
- Nhiều tab cùng mở giỏ hàng: khi 1 tab thêm/xóa item, cập nhật đồng bộ badge số lượng ở các tab khác qua `storage` event (hoặc BroadcastChannel), tránh tình trạng 1 tab ghi đè localStorage của tab khác làm mất thay đổi vừa thực hiện ở tab kia.
- Xử lý khi localStorage đầy (quota exceeded) hoặc bị user tắt qua chế độ duyệt riêng tư/tắt storage — có fallback (in-memory only, cảnh báo user) mà không crash trang.
- Dữ liệu form checkout dở dang (địa chỉ, note) có TTL tự hết hạn (ví dụ sau 24h) để tránh hiển thị lại thông tin đã lỗi thời (địa chỉ cũ, khuyến mãi hết hạn) như là dữ liệu hiện tại.

---

# Base ERD — Giỏ hàng và checkout trước khi có persistence bằng localStorage

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống **trước khi** giỏ hàng/form checkout được lưu vào localStorage. Đề bài giả định đã có sản phẩm, giỏ hàng, và checkout hoạt động server-driven cho user đã đăng nhập; còn giỏ hàng của guest (chưa đăng nhập) chỉ tồn tại tạm trong bộ nhớ JS của trang, mất ngay khi reload/đóng tab. Chưa có bất kỳ entity nào phía client (localStorage cart, snapshot giá, đồng bộ đa tab, TTL draft) — toàn bộ phần đó là enhance.

```mermaid
erDiagram
    USER ||--o| CART : "has (khi đã đăng nhập)"
    CART ||--o{ CART_ITEM : contains
    PRODUCT ||--o{ CART_ITEM : "referenced by"

    USER {
        string id PK
        string email
    }
    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
    }
    CART {
        string id PK
        string user_id FK
        string status "active | ordered"
    }
    CART_ITEM {
        string id PK
        string cart_id FK
        string product_id FK
        int quantity
        decimal price_at_add
    }
```

---

# Base sequence — Add to cart (guest cart chỉ tồn tại trong bộ nhớ, mất khi reload)

Đây là **base**, flow thêm sản phẩm vào giỏ ở trạng thái trước enhance. Với user đã đăng nhập, item được ghi thẳng lên giỏ hàng server. Với guest (chưa đăng nhập), giỏ hàng chỉ tồn tại trong biến JS ở bộ nhớ trang, không có bước lưu nào ra localStorage — nên reload hoặc đóng nhầm tab là mất toàn bộ giỏ. Đây chính là vấn đề mà enhance sẽ giải quyết bằng localStorage.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant Mem as In-memory cart (JS variable, guest)
    participant API as Cart API
    participant DB as Server DB

    User->>SPA: Bấm thêm sản phẩm vào giỏ
    alt Đã đăng nhập
        SPA->>API: POST thêm CART_ITEM
        API->>DB: Lưu CART_ITEM(product_id, quantity, price_at_add)
        API-->>SPA: Xác nhận, trả về giỏ hàng cập nhật
    else Guest (chưa đăng nhập)
        SPA->>Mem: Thêm item vào biến giỏ hàng tạm trong bộ nhớ
        Note over Mem: Không có bước lưu nào ra localStorage, chỉ tồn tại trong phiên trang hiện tại
        Mem-->>SPA: Cập nhật badge số lượng trên UI
    end
    Note over SPA,Mem: Nếu guest reload trang hoặc đóng nhầm tab, toàn bộ giỏ hàng trong bộ nhớ bị mất
```

---

# Base sequence — Checkout (form nhập trực tiếp, không lưu nháp, giá lấy live)

Đây là **base**, flow checkout ở trạng thái trước enhance. Người dùng điền địa chỉ/ghi chú trực tiếp trên form, không có bước lưu nháp nào — nếu thoát giữa chừng thì mất hết thông tin đã nhập. Giá sản phẩm được lấy trực tiếp từ CART_ITEM đã ghi ở bước thêm giỏ hàng, dùng luôn để đặt hàng, không có bước revalidate lại. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi thêm revalidate giá/tồn kho và cơ chế TTL cho form nháp.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Checkout UI
    participant API as Order API
    participant DB as Server DB

    User->>SPA: Mở trang checkout
    SPA->>API: Lấy CART_ITEM hiện tại (product, quantity, price_at_add)
    API-->>SPA: Trả về danh sách item và giá đã ghi từ lúc thêm giỏ
    User->>SPA: Điền địa chỉ, ghi chú trực tiếp trên form
    Note over SPA: Không có bước lưu nháp, nếu thoát giữa chừng sẽ mất thông tin đã nhập
    User->>SPA: Bấm đặt hàng
    SPA->>API: Submit order với giá price_at_add đã có, không kiểm tra lại giá/tồn kho hiện tại
    API->>DB: Tạo đơn hàng
    API-->>SPA: Xác nhận đặt hàng thành công
```

---

# Enhance ERD — Giỏ hàng và checkout draft qua localStorage

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (CART/CART_ITEM chỉ có ý nghĩa cho user đã đăng nhập, guest cart chỉ ở bộ nhớ), phần enhance thêm 5 nhóm entity mới ở phía client, ứng trực tiếp với các yêu cầu trong đề bài:

- `LOCAL_CART_ITEM` (mới, localStorage) — giỏ hàng guest được lưu bền thay vì chỉ ở bộ nhớ, kèm `price_snapshot` tại thời điểm thêm — nền cho yêu cầu 1 và 2.
- `CART_MERGE_LOG` (mới) — ghi lại quyết định merge giữa giỏ local và giỏ server khi đăng nhập (cộng dồn, giữ số lượng lớn hơn, hoặc hỏi user) — yêu cầu 1.
- `PRICE_REVALIDATION` (mới) — kết quả so sánh `price_snapshot` cục bộ với giá/tồn kho thật tại thời điểm checkout — yêu cầu 2.
- `CART_SYNC_EVENT` (mới) — sự kiện phát qua `storage`/BroadcastChannel khi 1 tab add/remove item, để tab khác cập nhật badge mà không ghi đè thay đổi của nhau — yêu cầu 3.
- `STORAGE_FALLBACK_STATE` (mới) — trạng thái localStorage có khả dụng không (quota exceeded hoặc bị chặn), có đang fallback in-memory không — yêu cầu 4.
- `CHECKOUT_DRAFT_LOCAL` (mới, localStorage) — form checkout dở dang (địa chỉ, note) kèm `expires_at` để tự hết hạn — yêu cầu 5.

```mermaid
erDiagram
    USER ||--o| CART : "has (khi đã đăng nhập)"
    CART ||--o{ CART_ITEM : contains
    PRODUCT ||--o{ CART_ITEM : "referenced by"
    PRODUCT ||--o{ LOCAL_CART_ITEM : "referenced by"
    LOCAL_CART_ITEM ||--o{ PRICE_REVALIDATION : "checked before checkout"
    LOCAL_CART_ITEM ||--o{ CART_SYNC_EVENT : "triggers"
    USER ||--o{ CART_MERGE_LOG : "merge history on login"
    USER ||--o| CHECKOUT_DRAFT_LOCAL : "has draft (localStorage)"
    USER ||--o| STORAGE_FALLBACK_STATE : "has per browser session"

    USER {
        string id PK
        string email
    }
    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
    }
    CART {
        string id PK
        string user_id FK
        string status "active | ordered"
    }
    CART_ITEM {
        string id PK
        string cart_id FK
        string product_id FK
        int quantity
        decimal price_at_add
    }
    LOCAL_CART_ITEM {
        string id PK
        string product_id FK
        int quantity
        decimal price_snapshot
        datetime added_at
        datetime updated_at
    }
    CART_MERGE_LOG {
        string id PK
        string user_id FK
        string product_id FK
        int local_quantity
        int server_quantity
        string resolution "sum | keep_max | asked_user"
        datetime merged_at
    }
    PRICE_REVALIDATION {
        string id PK
        string local_cart_item_id FK
        decimal price_snapshot
        decimal current_price
        int current_stock
        string status "unchanged | price_changed | out_of_stock"
        datetime checked_at
    }
    CART_SYNC_EVENT {
        string id PK
        string origin_tab_id
        string product_id FK
        string action "add | remove | update_qty"
        int quantity
        datetime broadcast_at
    }
    STORAGE_FALLBACK_STATE {
        string id PK
        string user_id FK
        string session_id
        boolean local_storage_available
        boolean fallback_in_memory_active
        datetime detected_at
    }
    CHECKOUT_DRAFT_LOCAL {
        string id PK
        string user_id FK
        string address
        string note
        datetime saved_at
        datetime expires_at
    }
```

---

# Enhance sequence — Add to cart (guest cart bền vững qua localStorage)

Đây là **enhance**, cùng flow "add-to-cart" như ở base nhưng thay đổi ở phần guest: thay vì chỉ giữ trong biến bộ nhớ JS (mất khi reload), item được ghi vào localStorage kèm `price_snapshot` tại thời điểm thêm. Đây là nền cho yêu cầu 1 (merge khi đăng nhập) và yêu cầu 2 (snapshot giá để revalidate khi checkout); việc phát sự kiện đồng bộ đa tab được nói rõ hơn ở flow multi-tab-cart-sync riêng.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant LS as localStorage (LOCAL_CART_ITEM)
    participant API as Cart API
    participant DB as Server DB

    User->>SPA: Bấm thêm sản phẩm vào giỏ
    alt Đã đăng nhập
        SPA->>API: POST thêm CART_ITEM
        API->>DB: Lưu CART_ITEM(product_id, quantity, price_at_add)
        API-->>SPA: Xác nhận, trả về giỏ hàng cập nhật
    else Guest (chưa đăng nhập)
        SPA->>LS: Ghi/cập nhật LOCAL_CART_ITEM(product_id, quantity, price_snapshot=giá hiện tại, updated_at)
        LS-->>SPA: Ghi thành công
        Note over LS: Giỏ hàng vẫn còn sau khi reload hoặc mở lại tab, khác với base
        SPA->>SPA: Phát sự kiện đồng bộ cho các tab khác (chi tiết ở flow multi-tab-cart-sync)
    end
```

---

# Enhance sequence — Merge giỏ hàng local với giỏ hàng server khi đăng nhập

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance. Khi user đăng nhập, giỏ hàng guest trong localStorage phải được merge đúng với giỏ hàng đã có sẵn trên server của tài khoản đó, theo quy tắc ưu tiên rõ ràng, không ghi đè ngầm định gây mất dữ liệu. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant LS as localStorage (LOCAL_CART_ITEM)
    participant API as Cart API
    participant DB as Server DB
    participant MergeLog as CART_MERGE_LOG

    User->>SPA: Đăng nhập thành công
    SPA->>LS: Đọc toàn bộ LOCAL_CART_ITEM hiện có (giỏ guest)
    SPA->>API: Lấy CART_ITEM hiện có trên server của tài khoản này
    API-->>SPA: Trả về giỏ hàng server

    loop Với mỗi sản phẩm xuất hiện ở cả 2 bên
        alt Số lượng khác nhau nhưng không mâu thuẫn nghiêm trọng
            SPA->>SPA: Áp quy tắc mặc định, ví dụ cộng dồn số lượng hoặc giữ số lượng lớn hơn
        else Chênh lệch lớn hoặc giá đã đổi giữa lúc thêm ở local và giá server hiện tại
            SPA-->>User: Hỏi rõ user chọn giữ giỏ local, giỏ server, hay cộng dồn
        end
        SPA->>MergeLog: Ghi CART_MERGE_LOG(local_quantity, server_quantity, resolution)
    end

    SPA->>API: Ghi kết quả merge cuối cùng lên CART_ITEM trên server
    API->>DB: Cập nhật giỏ hàng server theo kết quả merge
    API-->>SPA: Xác nhận giỏ hàng đã hợp nhất
    SPA->>LS: Xoá LOCAL_CART_ITEM vì đã merge xong lên server
```

---

# Enhance sequence — Checkout (revalidate giá/tồn kho trước khi thanh toán)

Đây là **enhance**, cùng flow "checkout" như ở base nhưng thêm bước revalidate: vì giá dùng ở giỏ hàng có thể là `price_snapshot`/`price_at_add` đã cũ (đặc biệt với guest cart lưu lâu trong localStorage), trước khi xác nhận thanh toán hệ thống phải kiểm tra lại giá/tồn kho thật với server và xử lý rõ khi giá đổi hoặc hết hàng, không để user thanh toán nhầm giá cũ. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Checkout UI
    participant API as Order API
    participant DB as Server DB
    participant Reval as PRICE_REVALIDATION

    User->>SPA: Mở trang checkout
    SPA->>SPA: Lấy danh sách item và price_snapshot/price_at_add hiện có
    User->>SPA: Điền/xác nhận địa chỉ, ghi chú
    User->>SPA: Bấm đặt hàng

    SPA->>API: Revalidate giá và tồn kho thật cho từng item trước khi submit order
    API->>DB: Lấy giá và stock hiện tại của từng sản phẩm
    DB-->>API: Trả về current_price, current_stock
    API-->>SPA: Trả kết quả so sánh với price_snapshot

    loop Với mỗi item
        alt Giá và tồn kho không đổi
            SPA->>Reval: Ghi PRICE_REVALIDATION status=unchanged
        else Giá đã thay đổi
            SPA->>Reval: Ghi PRICE_REVALIDATION status=price_changed
            SPA-->>User: Cảnh báo rõ giá đã thay đổi từ price_snapshot sang current_price, yêu cầu xác nhận lại
        else Hết hàng
            SPA->>Reval: Ghi PRICE_REVALIDATION status=out_of_stock
            SPA-->>User: Cảnh báo sản phẩm đã hết hàng, yêu cầu xoá khỏi giỏ hoặc chọn số lượng khác
        end
    end

    alt Toàn bộ item hợp lệ sau revalidate hoặc user đã xác nhận giá mới
        SPA->>API: Submit order với current_price đã revalidate
        API->>DB: Tạo đơn hàng
        API-->>SPA: Xác nhận đặt hàng thành công
    else Còn item chưa xử lý (hết hàng chưa gỡ, giá mới chưa xác nhận)
        SPA-->>User: Chặn đặt hàng cho tới khi xử lý xong các cảnh báo
    end
```

---

# Enhance sequence — Checkout draft tự hết hạn theo TTL

Đây là **enhance**, flow hoàn toàn mới, xử lý form checkout dở dang (địa chỉ, ghi chú) lưu trong localStorage. Vì thông tin này có thể lỗi thời (địa chỉ cũ, khuyến mãi hết hạn) nếu để lâu, mỗi bản nháp phải có TTL (ví dụ 24h) và tự động không được coi là dữ liệu hiện tại khi đã hết hạn. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Checkout UI
    participant LS as localStorage (CHECKOUT_DRAFT_LOCAL)

    User->>SPA: Điền địa chỉ, ghi chú trên form checkout
    SPA->>LS: Lưu CHECKOUT_DRAFT_LOCAL(address, note, saved_at, expires_at=saved_at+24h)
    Note over User,SPA: User thoát trang giữa chừng, chưa đặt hàng

    User->>SPA: Quay lại trang checkout sau đó
    SPA->>LS: Đọc CHECKOUT_DRAFT_LOCAL nếu có
    alt Còn hạn (now < expires_at)
        LS-->>SPA: Trả về address, note đã lưu
        SPA-->>User: Điền sẵn lại form với dữ liệu nháp còn hiệu lực
    else Đã hết hạn (now >= expires_at, ví dụ quá 24h)
        LS-->>SPA: Draft coi như không hợp lệ nữa
        SPA->>LS: Xoá CHECKOUT_DRAFT_LOCAL đã hết hạn
        SPA-->>User: Hiển thị form trống, không dùng địa chỉ/khuyến mãi cũ như dữ liệu hiện tại
    end
```

---

# Enhance sequence — Đồng bộ giỏ hàng giữa nhiều tab

Đây là **enhance**, flow hoàn toàn mới, xử lý trường hợp mở nhiều tab cùng giỏ hàng guest. Khi 1 tab thêm/xoá item, các tab khác phải cập nhật badge số lượng qua `storage` event hoặc BroadcastChannel, đồng thời tránh tình trạng 1 tab ghi đè toàn bộ localStorage của tab khác làm mất thay đổi vừa thực hiện ở tab kia. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor UserA as Người dùng (Tab A)
    participant TabA as SPA Tab A
    participant LS as localStorage (LOCAL_CART_ITEM, dùng chung origin)
    participant Channel as storage event / BroadcastChannel
    participant TabB as SPA Tab B

    UserA->>TabA: Thêm sản phẩm X vào giỏ ở Tab A
    TabA->>LS: Đọc LOCAL_CART_ITEM hiện tại, chỉ cập nhật đúng item của sản phẩm X, không ghi đè toàn bộ danh sách
    TabA->>LS: Ghi lại LOCAL_CART_ITEM đã cập nhật
    TabA->>Channel: Phát CART_SYNC_EVENT(action=add, product_id=X, quantity, origin_tab_id=A)

    Channel-->>TabB: Nhận CART_SYNC_EVENT từ Tab A
    TabB->>LS: Đọc lại LOCAL_CART_ITEM mới nhất từ localStorage, không dùng bản cache cũ trong bộ nhớ Tab B
    TabB-->>UserA: Cập nhật badge số lượng giỏ hàng ở Tab B khớp với thay đổi từ Tab A

    Note over TabA,TabB: Vì mỗi thao tác chỉ cập nhật đúng item liên quan rồi ghi lại toàn bộ state mới nhất đọc từ localStorage, thay đổi gần nhất ở tab này không bị tab kia ghi đè mất
```

---

# Enhance sequence — Fallback khi localStorage đầy hoặc bị chặn

Đây là **enhance**, flow hoàn toàn mới, xử lý trường hợp localStorage bị đầy (quota exceeded) hoặc bị user tắt (chế độ duyệt riêng tư/chặn storage). Hệ thống phải phát hiện tình trạng này, chuyển sang fallback in-memory only, và cảnh báo user mà không làm crash trang. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant LS as localStorage
    participant Mem as In-memory fallback cart
    participant Fallback as STORAGE_FALLBACK_STATE

    User->>SPA: Bấm thêm sản phẩm vào giỏ (guest)
    SPA->>LS: Thử ghi LOCAL_CART_ITEM

    alt Ghi thành công
        LS-->>SPA: OK
        SPA->>Fallback: Ghi local_storage_available=true, fallback_in_memory_active=false
    else Lỗi quota exceeded hoặc storage bị chặn (chế độ riêng tư)
        LS-->>SPA: Ném lỗi khi ghi (QuotaExceededError hoặc SecurityError)
        SPA->>SPA: Bắt lỗi, không để lỗi này làm crash trang
        SPA->>Mem: Chuyển sang giữ giỏ hàng tạm trong bộ nhớ (in-memory only) cho phiên hiện tại
        SPA->>Fallback: Ghi local_storage_available=false, fallback_in_memory_active=true
        SPA-->>User: Hiển thị cảnh báo, trình duyệt đang chặn lưu tạm nên giỏ hàng có thể mất khi đóng tab
    end

    Note over Mem: Trong chế độ fallback, giỏ hàng vẫn hoạt động bình thường trong phiên hiện tại nhưng không còn bền qua reload
```
