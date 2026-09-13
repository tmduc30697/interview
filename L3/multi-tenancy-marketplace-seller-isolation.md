# Cách ly dữ liệu bán hàng giữa các seller trên marketplace dùng chung nền tảng

**Hệ thống:** Một marketplace cho phép nhiều người bán (seller) độc lập đăng sản phẩm và bán hàng trên cùng một nền tảng dùng chung hạ tầng, mỗi seller có dữ liệu đơn hàng, doanh thu, và thông tin khách hàng của riêng họ.

**Vai trò của flow:** Cách ly ở đây phải đảm bảo một seller không bao giờ nhìn thấy dữ liệu bán hàng/doanh thu của seller khác dù họ đang cùng thao tác trên chung một giao diện quản lý (dashboard người bán) và chung hạ tầng backend, khác với mô hình một công ty một tenant thông thường vì số lượng seller có thể lên tới hàng chục nghìn và liên tục có seller mới tham gia/rời đi.

**Yêu cầu cụ thể:**
- Mọi truy vấn liên quan tới đơn hàng, doanh thu, tồn kho trên dashboard người bán phải bị giới hạn đúng theo `seller_id` ở tầng thấp nhất (không dựa vào việc ẩn nút bấm trên giao diện), kể cả với các API nội bộ dùng chung để tính toán số liệu tổng hợp cho nhiều mục đích khác nhau của nền tảng.
- Xử lý đúng trường hợp một đơn hàng có sản phẩm từ nhiều seller khác nhau trong cùng một giỏ hàng của khách mua — mỗi seller chỉ được xem phần đơn hàng liên quan tới sản phẩm của mình (thông tin vận chuyển, số lượng, giá của phần họ bán), không được thấy toàn bộ chi tiết đơn hàng gộp bao gồm phần của seller khác.
- Đảm bảo các báo cáo/thống kê xếp hạng do nền tảng cung cấp cho seller (ví dụ so sánh hiệu suất bán hàng với "mức trung bình ngành") không bị suy luận ngược ra số liệu cụ thể của một seller đối thủ cụ thể — đặc biệt khi ngành hàng đó chỉ có rất ít seller tham gia khiến số liệu trung bình gần như lộ số liệu của một seller cụ thể.
- Xử lý trường hợp một seller ngừng hoạt động/bị khóa tài khoản (vi phạm chính sách hoặc tự nguyện rời sàn) — dữ liệu lịch sử đơn hàng liên quan tới khách mua vẫn cần được giữ cho mục đích hỗ trợ khách hàng và pháp lý, nhưng seller đã rời sàn không được tiếp tục truy cập vào dữ liệu đó, và các seller khác cũng không được vô tình thấy dữ liệu của seller đã rời sàn qua các báo cáo tổng hợp toàn nền tảng.
- Đảm bảo tính năng chăm sóc khách hàng của nền tảng (nhân viên marketplace hỗ trợ khách mua) khi cần tra cứu một đơn hàng cụ thể chỉ được xem đúng phạm vi dữ liệu liên quan tới đơn hàng đó, không được có quyền truy cập mặc định vào toàn bộ dữ liệu kinh doanh của seller liên quan, và mọi lần tra cứu phải được ghi log lại.

---

# Base ERD — Marketplace trước khi siết cách ly seller ở tầng dữ liệu

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** siết cách ly. Đề bài đối chiếu trực tiếp với việc hiện tại chỉ "ẩn nút bấm trên giao diện" — nên base được suy luận là một marketplace đã có đủ: seller, sản phẩm, đơn hàng có thể gộp nhiều seller trong cùng giỏ hàng (`ORDER_ITEM.seller_id` đã tồn tại vì đây là cấu trúc tự nhiên của giỏ hàng đa seller, không phải điều enhance mới thêm), và một báo cáo benchmark ngành đã có sẵn nhưng tính trung bình ngây thơ không xét cỡ mẫu. Base **chưa có** cơ chế enforce `seller_id` ở tầng data/query layer (chỉ dựa vào UI ẩn), **chưa có** ngưỡng mẫu tối thiểu cho benchmark, **chưa có** quy trình offboarding seller giữ lịch sử nhưng thu hồi quyền truy cập, và **chưa có** log tra cứu đơn hàng có phạm vi giới hạn cho nhân viên hỗ trợ — những phần này là enhance.

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "ordered as"
    SELLER ||--o{ ORDER_ITEM : sells
    INDUSTRY_BENCHMARK_REPORT ||--o{ ORDER_ITEM : "computed from (cùng category)"

    SELLER {
        string id PK
        string name
        string status "active"
        datetime joined_at
    }
    BUYER {
        string id PK
        string name
        string email
    }
    PRODUCT {
        string id PK
        string seller_id FK
        string category
        string title
        decimal price
        int stock
    }
    ORDER {
        string id PK
        string buyer_id FK
        datetime created_at
        string status
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        string seller_id FK "denormalized từ product, phục vụ giỏ hàng đa seller"
        int quantity
        decimal unit_price
        string shipping_status
    }
    INDUSTRY_BENCHMARK_REPORT {
        string id PK
        string category
        decimal avg_revenue "tính trung bình ngây thơ, không xét cỡ mẫu"
        datetime computed_at
    }
```

---

# Base sequence — Xem dashboard người bán (lọc ở tầng UI, không phải tầng data)

Đây là **base**, flow "Xem dashboard người bán" ở trạng thái hiện tại — API nội bộ dùng chung cho nhiều mục đích của nền tảng trả về dữ liệu chưa được giới hạn chặt theo `seller_id`, việc chỉ hiển thị đúng dữ liệu của seller đang đăng nhập được xử lý ở tầng giao diện. Flow này liên quan mật thiết tới enhance vì yêu cầu 1 của đề bài chính là chuyển việc giới hạn này xuống tầng thấp nhất thay vì dựa vào UI.

```mermaid
sequenceDiagram
    actor Seller as Seller A
    participant App as Seller Dashboard App
    participant Agg as Internal Aggregate API (dùng chung nhiều mục đích)
    participant DB as ORDER_ITEM store

    Seller->>App: Xem đơn hàng/doanh thu/tồn kho của mình
    App->>Agg: Gọi API tổng hợp số liệu theo category/thời gian
    Agg->>DB: Query ORDER_ITEM trong khoảng thời gian, theo category (không lọc seller_id ở query)
    DB-->>Agg: Trả về toàn bộ ORDER_ITEM liên quan, gồm cả của seller khác
    Agg-->>App: Trả về tập dữ liệu chưa giới hạn theo seller
    App->>App: Lọc lại ở tầng giao diện, chỉ hiển thị hàng có seller_id = Seller A
    App-->>Seller: Hiển thị dashboard (đã lọc ở UI)
    Note over App,Agg: Nếu UI lọc sai hoặc client tự gọi thẳng Internal Aggregate API, dữ liệu seller khác bị lộ vì tầng dưới không hề enforce seller_id
```

---

# Base sequence — Xem chi tiết đơn hàng đa seller (lộ toàn bộ đơn hàng)

Đây là **base**, flow "Xem chi tiết đơn hàng" ở trạng thái hiện tại — khi một đơn hàng có sản phẩm từ nhiều seller, seller nào mở đơn hàng cũng thấy toàn bộ chi tiết gộp, kể cả phần của seller khác. Flow này liên quan mật thiết tới enhance vì yêu cầu 2 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    actor SellerA as Seller A
    participant App as Order Detail Service
    participant DB as ORDER / ORDER_ITEM store

    SellerA->>App: Xem chi tiết đơn hàng O (có sản phẩm của Seller A và Seller B)
    App->>DB: SELECT * FROM order_item WHERE order_id = O
    DB-->>App: Toàn bộ ORDER_ITEM của đơn O, gồm cả của Seller B
    App-->>SellerA: Hiển thị toàn bộ đơn hàng, gồm số lượng/giá/vận chuyển của cả Seller B
    Note over App,DB: Seller A nhìn thấy giá bán, số lượng, thông tin vận chuyển của sản phẩm thuộc Seller B — lộ dữ liệu kinh doanh của đối thủ
```

---

# Base sequence — Xem benchmark ngành (trung bình ngây thơ, có thể suy luận ngược)

Đây là **base**, flow "Xem báo cáo so sánh hiệu suất với trung bình ngành" ở trạng thái hiện tại — tính trung bình doanh thu toàn bộ seller trong category mà không xét số lượng seller tham gia. Flow này liên quan mật thiết tới enhance vì yêu cầu 3 của đề bài chính là chặn kiểu suy luận ngược này.

```mermaid
sequenceDiagram
    actor SellerA as Seller A (ngành hàng chỉ có 2 seller)
    participant App as Benchmark Service
    participant DB as INDUSTRY_BENCHMARK_REPORT

    SellerA->>App: Xem so sánh doanh thu với "mức trung bình ngành"
    App->>DB: Lấy avg_revenue của category (tính trên toàn bộ seller trong category, không xét cỡ mẫu)
    DB-->>App: avg_revenue (tính trên đúng 2 seller: A và B)
    App-->>SellerA: Hiển thị "Doanh thu trung bình ngành: X"
    Note over SellerA: Seller A tự biết doanh thu của mình, suy ra ngay doanh thu Seller B = 2*X - doanh_thu_A — số liệu đối thủ bị lộ hoàn toàn
```

---

# Enhance ERD — sau khi siết cách ly seller ở tầng dữ liệu

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 3 nhóm thay đổi/entity mới, ứng trực tiếp với yêu cầu 3, 4, 5 trong đề bài (yêu cầu 1 và 2 là thay đổi ở tầng enforcement/query logic trên đúng các entity đã có sẵn — `ORDER_ITEM.seller_id` vẫn là chỗ dựa để lọc, không cần entity mới):

- `SELLER` (sửa) — mở rộng `status` thêm `suspended | offboarded` và `deactivated_at`, phục vụ yêu cầu 4.
- `INDUSTRY_BENCHMARK_REPORT` (sửa) — thêm `seller_count_in_sample`, `min_sample_size`, `suppressed` để chặn suy luận ngược khi cỡ mẫu quá nhỏ (yêu cầu 3), đồng thời vẫn tính đúng khi có seller đã offboard (yêu cầu 4).
- `SELLER_OFFBOARDING_EVENT` (mới) — ghi nhận lý do và thời điểm seller rời sàn, phục vụ thu hồi quyền truy cập trong khi giữ lại dữ liệu lịch sử cho hỗ trợ khách hàng/pháp lý.
- `SUPPORT_ORDER_LOOKUP_LOG` (mới) — log mọi lần nhân viên chăm sóc khách hàng tra cứu một đơn hàng cụ thể, phục vụ yêu cầu 5.

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "ordered as"
    SELLER ||--o{ ORDER_ITEM : sells
    INDUSTRY_BENCHMARK_REPORT ||--o{ ORDER_ITEM : "computed from (cùng category, có ngưỡng mẫu tối thiểu)"
    SELLER ||--o{ SELLER_OFFBOARDING_EVENT : "may have"
    ORDER ||--o{ SUPPORT_ORDER_LOOKUP_LOG : "looked up via"

    SELLER {
        string id PK
        string name
        string status "active | suspended | offboarded"
        datetime joined_at
        datetime deactivated_at "nullable"
    }
    BUYER {
        string id PK
        string name
        string email
    }
    PRODUCT {
        string id PK
        string seller_id FK
        string category
        string title
        decimal price
        int stock
    }
    ORDER {
        string id PK
        string buyer_id FK
        datetime created_at
        string status
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        string seller_id FK "chỗ dựa để enforce ở tầng data layer"
        int quantity
        decimal unit_price
        string shipping_status
    }
    INDUSTRY_BENCHMARK_REPORT {
        string id PK
        string category
        decimal avg_revenue
        int seller_count_in_sample
        int min_sample_size "vd 5"
        boolean suppressed "true nếu seller_count_in_sample < min_sample_size"
        datetime computed_at
    }
    SELLER_OFFBOARDING_EVENT {
        string id PK
        string seller_id FK
        string reason "policy_violation | voluntary"
        datetime offboarded_at
        string data_retention_note
    }
    SUPPORT_ORDER_LOOKUP_LOG {
        string id PK
        string support_staff_id
        string order_id FK
        string reason
        datetime looked_up_at
    }
```

---

# Enhance sequence — Xem dashboard người bán (enforce seller_id ở tầng data layer)

Đây là **enhance**, cùng flow "Xem dashboard người bán" đã có ở base nhưng nay thay đổi theo yêu cầu 1 của đề bài: `seller_id` được enforce ngay tại tầng data/query layer — kể cả Internal Aggregate API dùng chung cho nhiều mục đích khác của nền tảng — thay vì chỉ lọc ở UI như base.

```mermaid
sequenceDiagram
    actor Seller as Seller A
    participant App as Seller Dashboard App
    participant Agg as Internal Aggregate API (dùng chung nhiều mục đích)
    participant Scope as Query Scoping Layer (bắt buộc mọi caller)
    participant DB as ORDER_ITEM store

    Seller->>App: Xem đơn hàng/doanh thu/tồn kho của mình
    App->>Agg: Gọi API tổng hợp số liệu, kèm seller_id=A trong security context
    Agg->>Scope: Chuyển query xuống, bắt buộc đi qua Query Scoping Layer
    Scope->>Scope: Tiêm điều kiện WHERE seller_id = A vào mọi query, không cho phép bỏ qua
    Scope->>DB: Query ORDER_ITEM đã bị giới hạn seller_id = A ngay tại tầng data
    DB-->>Scope: Chỉ trả ORDER_ITEM của Seller A
    Scope-->>Agg: Dữ liệu đã an toàn
    Agg-->>App: Trả về đúng phạm vi Seller A
    App-->>Seller: Hiển thị dashboard

    Note over Scope: Mọi caller khác của Internal Aggregate API (vd tool phân tích nội bộ, job vận hành) cũng đi qua cùng Query Scoping Layer, muốn truy vấn xuyên seller phải có context đặc quyền riêng, tự nó cũng bị audit
```

---

# Enhance sequence — Xem chi tiết đơn hàng đa seller (chỉ thấy phần của mình)

Đây là **enhance**, cùng flow "Xem chi tiết đơn hàng" đã có ở base nhưng nay thay đổi theo yêu cầu 2 của đề bài: mỗi seller tham gia một đơn hàng chỉ xem được `ORDER_ITEM` (số lượng, giá, vận chuyển) thuộc về sản phẩm của chính mình, không thấy toàn bộ đơn hàng gộp như base.

```mermaid
sequenceDiagram
    actor SellerA as Seller A
    participant App as Order Detail Service
    participant DB as ORDER / ORDER_ITEM store

    SellerA->>App: Xem chi tiết đơn hàng O (có sản phẩm của Seller A và Seller B)
    App->>DB: SELECT * FROM order_item WHERE order_id = O AND seller_id = A
    DB-->>App: Chỉ ORDER_ITEM thuộc Seller A trong đơn O
    App-->>SellerA: Hiển thị đúng phần của Seller A, kèm mã đơn hàng chung để đối chiếu với khách mua nhưng không có chi tiết phần của Seller B
    Note over App,DB: Nếu đơn hàng có nhiều seller, mỗi seller chỉ nhận view riêng của phần mình, không có API nào trả gộp toàn bộ item cho một seller thường
```

---

# Enhance sequence — Xem benchmark ngành (chặn suy luận ngược khi cỡ mẫu nhỏ)

Đây là **enhance**, cùng flow "Xem báo cáo so sánh hiệu suất với trung bình ngành" đã có ở base nhưng nay thay đổi theo yêu cầu 3 của đề bài: trước khi trả số liệu so sánh cụ thể, hệ thống kiểm tra `seller_count_in_sample` so với `min_sample_size`, nếu ngành hàng quá ít seller thì không tiết lộ số liệu chi tiết để tránh suy luận ngược ra đối thủ cụ thể.

```mermaid
sequenceDiagram
    actor SellerA as Seller A (ngành hàng chỉ có 2 seller)
    participant App as Benchmark Service
    participant DB as INDUSTRY_BENCHMARK_REPORT

    SellerA->>App: Xem so sánh doanh thu với "mức trung bình ngành"
    App->>DB: Lấy avg_revenue, seller_count_in_sample, min_sample_size của category
    DB-->>App: avg_revenue, seller_count_in_sample=2, min_sample_size=5

    alt seller_count_in_sample >= min_sample_size
        App-->>SellerA: Hiển thị số liệu so sánh cụ thể
    else seller_count_in_sample < min_sample_size
        App->>DB: Đánh dấu suppressed=true cho report này
        App-->>SellerA: Hiển thị thông báo chung "Chưa đủ dữ liệu ngành để so sánh chi tiết", không lộ avg_revenue cụ thể
        Note over App: Với cỡ mẫu quá nhỏ, dù chỉ đưa ra 1 con số trung bình cũng đủ để suy luận ngược ra đối thủ duy nhất còn lại — nên phải generalize hoặc gộp thêm category liên quan thay vì hiển thị số cụ thể
    end
```

---

# Enhance sequence — Seller offboarding (thu hồi quyền, giữ lịch sử, không lộ qua báo cáo tổng hợp)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có quy trình xử lý khi seller ngừng hoạt động, chỉ có `status = active` cố định. Đáp ứng yêu cầu 4 của đề bài: dữ liệu lịch sử đơn hàng vẫn giữ cho hỗ trợ khách hàng/pháp lý, seller đã rời sàn mất quyền truy cập, và các seller khác không vô tình thấy dữ liệu của seller đã rời qua báo cáo tổng hợp toàn nền tảng.

```mermaid
sequenceDiagram
    actor Ops as Vận hành nền tảng
    participant App as Seller Management Service
    participant SellerDB as SELLER store
    participant Event as SELLER_OFFBOARDING_EVENT
    participant OrderDB as ORDER / ORDER_ITEM store
    participant Bench as INDUSTRY_BENCHMARK_REPORT
    actor OffSeller as Seller đã rời sàn
    actor OtherSeller as Seller khác vẫn hoạt động

    Ops->>App: Khóa/offboard Seller X (vi phạm chính sách hoặc tự nguyện)
    App->>SellerDB: Cập nhật SELLER(X).status=offboarded, deactivated_at=now
    App->>Event: Tạo SELLER_OFFBOARDING_EVENT(seller_id=X, reason, data_retention_note)
    Note over OrderDB: ORDER/ORDER_ITEM lịch sử của Seller X không bị xóa, vẫn giữ nguyên cho mục đích hỗ trợ khách mua và pháp lý

    OffSeller->>App: Cố đăng nhập lại dashboard
    App->>SellerDB: Kiểm tra status
    SellerDB-->>App: status=offboarded
    App-->>OffSeller: Từ chối truy cập, tài khoản đã bị thu hồi quyền

    OtherSeller->>App: Xem benchmark ngành (category có cả Seller X trong lịch sử)
    App->>Bench: Tính avg_revenue có gộp dữ liệu lịch sử của Seller X, kiểm tra seller_count_in_sample >= min_sample_size như flow benchmark
    Bench-->>App: Kết quả đã qua kiểm tra ngưỡng mẫu tối thiểu
    App-->>OtherSeller: Hiển thị số liệu tổng hợp an toàn, không có cách nào tách riêng ra đóng góp cụ thể của Seller X
```

---

# Enhance sequence — Nhân viên chăm sóc khách hàng tra cứu đơn hàng (phạm vi giới hạn, có log)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có flow riêng cho nhân viên chăm sóc khách hàng, ngụ ý trước đó không có ranh giới rõ giữa quyền hỗ trợ và toàn bộ dữ liệu kinh doanh seller. Đáp ứng yêu cầu 5 của đề bài: khi tra cứu một đơn hàng cụ thể để hỗ trợ khách mua, nhân viên chỉ thấy đúng phạm vi đơn hàng đó, không có quyền mặc định vào toàn bộ dữ liệu kinh doanh của seller liên quan, và mọi lần tra cứu đều được ghi log.

```mermaid
sequenceDiagram
    actor Support as Nhân viên chăm sóc khách hàng
    participant App as Customer Support Service
    participant Log as SUPPORT_ORDER_LOOKUP_LOG
    participant DB as ORDER / ORDER_ITEM store

    Support->>App: Tra cứu đơn hàng O theo yêu cầu hỗ trợ của khách mua
    App->>Log: Ghi SUPPORT_ORDER_LOOKUP_LOG(support_staff_id, order_id=O, reason, looked_up_at)
    App->>DB: SELECT * FROM order_item WHERE order_id = O
    DB-->>App: Toàn bộ ORDER_ITEM thuộc đơn O (có thể gồm nhiều seller, cần thiết để hỗ trợ khách mua)
    App-->>Support: Hiển thị đúng phạm vi đơn hàng O

    Support->>App: Thử xem thêm các đơn hàng khác hoặc số liệu doanh thu tổng của seller liên quan
    App->>App: Kiểm tra quyền, support role không có scope ngoài đơn hàng đang xử lý
    App-->>Support: Từ chối, không có quyền mặc định vào toàn bộ dữ liệu kinh doanh của seller
    Note over Log: Mọi lần tra cứu đều để lại log tra soát được, kể cả các lần bị từ chối mở rộng phạm vi
```
