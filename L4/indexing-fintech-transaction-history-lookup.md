# Fintech tra cứu lịch sử giao dịch trên bảng transaction khổng lồ

**Hệ thống:** Ngân hàng số/ví điện tử có bảng `transactions` append-only với hàng trăm triệu dòng, phục vụ cả user tra cứu lịch sử giao dịch của mình lẫn CSKH/vận hành tra cứu theo nhiều tiêu chí khác nhau.

**Vai trò của flow:** Đảm bảo tra cứu nhanh và chính xác trong khi bảng vẫn đang bị ghi liên tục với tốc độ cao (giao dịch mới insert từng giây), mà không phá vỡ yêu cầu audit trail toàn vẹn.

**Yêu cầu cụ thể:**
- User tra lịch sử giao dịch của tài khoản mình cần composite index (account_id, created_at DESC) để lấy nhanh giao dịch gần nhất, nhưng CSKH lại cần lọc thêm theo status hoặc transaction_type trên cùng bảng — quyết định rõ có thêm cột vào index hiện tại, tạo index riêng, hay chấp nhận index intersection cho các trường hợp ít gặp hơn.
- Bảng append-only tăng kích thước liên tục khiến index B-tree ngày càng phình to, làm chậm dần cả tốc độ ghi lẫn thời gian bảo trì index — cần cân nhắc range-partition bảng theo tháng/năm (dựa trên created_at) kết hợp local index riêng từng partition để giữ mỗi index luôn nhỏ gọn.
- Truy vấn "toàn bộ giao dịch failed/pending" chỉ chiếm tỉ lệ rất nhỏ trên tổng bảng nhưng được CSKH chạy thường xuyên để xử lý sự cố — index toàn bảng theo status sẽ lãng phí vì đa số giao dịch đã success; cần dùng partial index (chỉ index các dòng có status IN pending/failed) để index nhỏ và nhanh hơn hẳn.
- Bộ phận compliance cần chạy query audit full-scan theo nhiều tổ hợp cột không thể index hết (đổi tiêu chí liên tục theo từng đợt kiểm toán) — cần quyết định có tách hẳn read replica riêng cho audit/reporting để không cạnh tranh tài nguyên index/buffer pool với luồng giao dịch thời gian thực.
- Giờ cao điểm có rất nhiều giao dịch insert đồng thời, index có cột đơn điệu tăng dần (như created_at hoặc id tự tăng) dễ tạo hot page ở cuối B-tree gây lock contention khi nhiều transaction cùng ghi — cần xem xét chiến lược giảm contention (vd tách theo shard/account_id trước khi index theo thời gian).

---

# ERD - Base: bảng transactions cơ bản, chưa tối ưu index/partition

Đây là **base**, trạng thái dữ liệu ngay trước khi áp đề bài. Hệ thống ngân hàng số/ví điện tử đã có `ACCOUNT` và bảng `TRANSACTION` append-only lưu mọi giao dịch, nhưng bảng chưa được partition và chỉ có index tối thiểu (khóa chính trên `transaction_id`, index đơn cột trên `account_id`) - đây chính là điểm xuất phát mà đề bài (composite index, partial index, partition, read replica) cần cải thiện.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "owns"

    ACCOUNT {
        string account_id PK
        string customer_id
        string account_type
        string currency
        string status
        datetime created_at
    }

    TRANSACTION {
        string transaction_id PK
        string account_id FK
        datetime created_at
        string status
        string transaction_type
        decimal amount
        string currency
        string description
    }

    %% Index hien co (khong the hien truc tiep trong ERD):
    %% - PRIMARY KEY btree tren transaction_id
    %% - idx_transactions_account_id: index don cot tren account_id
    %% - KHONG co composite index (account_id, created_at)
    %% - KHONG co partial index theo status
    %% - Bang TRANSACTION la 1 heap duy nhat, chua partition theo thoi gian
```

---

# Sequence - Base: User tra cứu lịch sử giao dịch

Đây là **base**, flow "user tra cứu lịch sử giao dịch của tài khoản mình". Flow này được chọn vì đây chính là truy vấn mà đề bài muốn tối ưu bằng composite index `(account_id, created_at DESC)` - ở base, bảng chỉ có index đơn cột trên `account_id` nên DB phải tự sort kết quả, minh hoạ rõ vấn đề cần giải quyết.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile/Web App
    participant API as Transaction API
    participant DB as Postgres - transactions

    User->>App: Mở màn hình lịch sử giao dịch
    App->>API: GET /accounts/{account_id}/transactions?page=1
    API->>DB: SELECT * FROM transactions WHERE account_id = $1 ORDER BY created_at DESC LIMIT 20

    Note over DB: Planner dung idx_transactions_account_id (index don cot)
    Note over DB: de tim cac dong khop account_id
    Note over DB: nhung index nay KHONG luu theo thu tu created_at
    Note over DB: nen phai them buoc sort (filesort) truoc khi LIMIT 20

    DB-->>API: 20 dòng giao dịch gần nhất (sau khi sort)
    API-->>App: Danh sách giao dịch
    App-->>User: Hiển thị lịch sử giao dịch

    Note over DB: Bang cang lon (hang tram trieu dong),<br/>chi phi sort cho moi truy van cang tang,<br/>du chi lay 20 dong dau
```

---

# Sequence - Base: CSKH lọc giao dịch failed/pending

Đây là **base**, flow "CSKH tra cứu toàn bộ giao dịch failed/pending để xử lý sự cố". Flow này được chọn vì đề bài đề xuất partial index cho đúng use case này - ở base, bảng chưa có bất kỳ index nào trên cột `status`, nên minh hoạ vấn đề: dù số dòng failed/pending chỉ là phần rất nhỏ, DB vẫn phải quét toàn bộ bảng.

```mermaid
sequenceDiagram
    actor CSKH as CSKH Agent
    participant Tool as Internal Ops Tool
    participant DB as Postgres - transactions

    CSKH->>Tool: Xem danh sách giao dịch failed/pending
    Tool->>DB: SELECT * FROM transactions WHERE status IN ('failed','pending') ORDER BY created_at DESC LIMIT 100

    Note over DB: Khong co index nao tren cot status
    Note over DB: Planner phai Seq Scan toan bo bang transactions
    Note over DB: (hang tram trieu dong) de tim ra so it dong khop dieu kien

    DB-->>Tool: 100 dòng failed/pending (sau khi quét toàn bảng)
    Tool-->>CSKH: Danh sách giao dịch cần xử lý

    Note over DB: Query nay chay thuong xuyen nhung rat ton I/O,<br/>vi phai doc ca cac dong status = success<br/>chiem da so bang de loc ra phan nho con lai
```

---

# Sequence - Base: Insert giao dịch mới (append-only)

Đây là **base**, flow "hệ thống thanh toán insert giao dịch mới vào bảng transactions". Flow này được chọn vì đề bài nhắc tới 2 vấn đề trực tiếp liên quan tới insert: (1) B-tree của bảng đơn (chưa partition) ngày càng phình to làm chậm ghi và bảo trì index, (2) giờ cao điểm insert đồng thời vào cột tăng dần (created_at/id) gây hot page ở cuối B-tree.

```mermaid
sequenceDiagram
    participant Pay as Payment Service
    participant DB as Postgres - transactions

    Pay->>DB: INSERT INTO transactions (transaction_id, account_id, created_at, status, ...)

    Note over DB: Chi co 1 bang transactions duy nhat (chua partition)
    Note over DB: Moi insert phai cap nhat CA HAI index tren toan bang:
    Note over DB: PRIMARY KEY (transaction_id) va idx_transactions_account_id

    Note over DB: transaction_id/created_at tang dan deu,<br/>nen cac insert moi lien tuc roi vao cung<br/>1 trang (leaf page) cuoi cung cua B-tree

    Note over DB: Gio cao diem: nhieu transaction insert dong thoi<br/>tranh chap lock/ghi tren cung hot page do

    DB-->>Pay: Insert thành công (nhưng chậm dần khi bảng lớn lên)

    Note over DB: Kich thuoc bang va index chi tang, khong co<br/>co che nao giu index nho gon theo thoi gian
```

---

# ERD - Enhance: transactions range-partition + partial index + read replica

Đây là **enhance**, đúng nguyên văn nội dung đề bài. So với base, bảng `TRANSACTION` logic nay được range-partition theo tháng/năm dựa trên `created_at`, mỗi partition mang bộ index riêng nhỏ gọn (composite `(account_id, created_at DESC)` cho user, partial index theo `status` cho CSKH), và có thêm `READ_REPLICA` tách riêng cho compliance/audit để không cạnh tranh tài nguyên với luồng giao dịch thời gian thực.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "owns"
    TRANSACTION ||--|{ TRANSACTION_PARTITION : "partitioned by month/year (created_at)"
    TRANSACTION ||--o{ READ_REPLICA : "streaming replication"

    ACCOUNT {
        string account_id PK
        string customer_id
        string account_type
        string currency
        string status
        datetime created_at
    }

    TRANSACTION {
        string transaction_id PK
        string account_id FK
        datetime created_at "partition key"
        string status
        string transaction_type
        decimal amount
        string currency
        string description
    }

    TRANSACTION_PARTITION {
        string partition_name PK
        date range_start
        date range_end
        string local_composite_index "btree (account_id, created_at DESC)"
        string local_partial_index "btree (status) WHERE status IN (pending,failed)"
    }

    READ_REPLICA {
        string replica_id PK
        string source_table FK
        string purpose "compliance/audit full-scan queries"
        string replication_lag
    }

    %% Quyet dinh thiet ke tu de bai:
    %% - KHONG them status/transaction_type vao composite (account_id, created_at)
    %%   de giu index chinh cho user gon nhe, CSKH loc status/type dung index rieng
    %%   hoac chap nhan index intersection cho truong hop it gap
    %% - Partial index tren status chi chua rows pending/failed (~ ty le nho)
    %% - Moi partition co bo index rieng, nho gon, khong phinh to theo thoi gian
    %% - Insert theo thoi gian van co the tao hot page trong 1 partition,
    %%   can them chien luoc giam contention (vd shard/hash theo account_id)
```

---

# Sequence - Enhance: User tra cứu lịch sử giao dịch

Đây là **enhance** của flow "user tra cứu lịch sử giao dịch" đã có ở base (file `2-base-user-transaction-history-lookup-sqd.md`). So với base, DB không còn phải filesort: bảng đã range-partition theo `created_at` nên planner prune bớt partition không liên quan, rồi dùng local composite index `(account_id, created_at DESC)` của từng partition còn lại để lấy dữ liệu đã sẵn đúng thứ tự.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile/Web App
    participant API as Transaction API
    participant DB as Postgres - transactions (partitioned)

    User->>App: Mở màn hình lịch sử giao dịch
    App->>API: GET /accounts/{account_id}/transactions?page=1
    API->>DB: SELECT * FROM transactions WHERE account_id = $1 ORDER BY created_at DESC LIMIT 20

    Note over DB: Partition pruning: chi can xet cac partition<br/>thang/nam gan nhat (nhung partition cu hon<br/>bi loai truoc khi quet)

    loop Với mỗi partition còn lại (mới nhất trước)
        Note over DB: Dung local index (account_id, created_at DESC)<br/>cua partition do - vua loc account_id<br/>vua tra ve dung thu tu created_at DESC
        DB-->>DB: Index scan tra ve dong da sort san, KHONG can filesort
    end

    DB-->>API: 20 dòng giao dịch gần nhất (gộp từ các partition, không filesort)
    API-->>App: Danh sách giao dịch
    App-->>User: Hiển thị lịch sử giao dịch

    Note over DB: So voi base: khong con buoc sort rieng,<br/>va moi partition co index nho gon nen<br/>toc do on dinh du bang tong the ngay cang lon
```

---

# Sequence - Enhance: CSKH lọc giao dịch failed/pending

Đây là **enhance** của flow "CSKH lọc giao dịch failed/pending" đã có ở base (file `3-base-cskh-filter-failed-pending-sqd.md`). So với base, mỗi partition nay có sẵn partial index chỉ chứa các dòng `status IN (pending, failed)`, nên thay vì Seq Scan toàn bảng, DB chỉ cần Bitmap Index Scan trên partial index rất nhỏ của từng partition.

```mermaid
sequenceDiagram
    actor CSKH as CSKH Agent
    participant Tool as Internal Ops Tool
    participant DB as Postgres - transactions (partitioned)

    CSKH->>Tool: Xem danh sách giao dịch failed/pending
    Tool->>DB: SELECT * FROM transactions WHERE status IN ('failed','pending') ORDER BY created_at DESC LIMIT 100

    loop Với mỗi partition
        Note over DB: Dung local partial index<br/>btree (status) WHERE status IN (pending,failed)
        Note over DB: Index nay chi chua vai % so dong cua partition,<br/>nen Bitmap Index Scan rat nhanh
    end

    DB-->>Tool: 100 dòng failed/pending (không cần quét dòng success)
    Tool-->>CSKH: Danh sách giao dịch cần xử lý

    Note over DB: So voi base: khong con Seq Scan toan bang,<br/>chi phi query gan nhu ty le voi so dong<br/>pending/failed thuc te, khong phu thuoc<br/>tong kich thuoc bang
```

---

# Sequence - Enhance: Insert giao dịch mới (partition + giảm hot page contention)

Đây là **enhance** của flow "insert giao dịch mới" đã có ở base (file `4-base-insert-transaction-sqd.md`). So với base, insert nay chỉ cần cập nhật index của đúng 1 partition (nhỏ gọn hơn nhiều so với index toàn bảng cũ), và có thêm bước phân tán ghi theo `account_id` để giảm tình trạng nhiều transaction cùng tranh chấp một hot page ở cuối B-tree trong giờ cao điểm.

```mermaid
sequenceDiagram
    participant Pay as Payment Service
    participant Router as Partition/Write Router
    participant DB as Postgres - transactions (partitioned)

    Pay->>Router: INSERT transaction (account_id, created_at, status, ...)
    Router->>Router: Xác định partition đích theo created_at (tháng/năm hiện tại)

    Note over Router: Ket hop them buoc phan tan ghi theo hash(account_id)<br/>de cac insert cung thoi diem khong don het<br/>vao 1 vung leaf page duy nhat cua partition

    Router->>DB: INSERT vào đúng partition (vd transactions_y2026m09)

    Note over DB: Chi 2 index CUC BO cua partition nay can cap nhat:<br/>composite (account_id, created_at DESC) va partial index status
    Note over DB: Index nho hon nhieu so voi index toan bang o base,<br/>nen chi phi maintain B-tree thap hon

    DB-->>Router: Insert thành công
    Router-->>Pay: Ack

    Note over DB: So voi base: (1) index can cap nhat nho gon<br/>hon vi chi thuoc 1 partition, (2) phan tan ghi<br/>theo account_id giam tranh chap hot page<br/>so voi ghi tuan tu thuan theo thoi gian
```

---

# Sequence - Enhance: Compliance chạy audit query trên read replica

Đây là **enhance**, flow hoàn toàn mới phát sinh từ đề bài, chưa tồn tại ở base. Bộ phận compliance cần chạy các query full-scan với tổ hợp cột thay đổi liên tục theo từng đợt kiểm toán, không thể index hết - flow này tách các truy vấn đó sang read replica riêng để không cạnh tranh index/buffer pool với luồng giao dịch thời gian thực (OLTP) trên primary.

```mermaid
sequenceDiagram
    actor Compliance as Compliance Analyst
    participant Tool as Audit/Reporting Tool
    participant Replica as Postgres Read Replica
    participant Primary as Postgres Primary - transactions

    Primary-->>Replica: Streaming replication liên tục (dữ liệu transactions)

    Compliance->>Tool: Chọn tiêu chí audit đợt này (tổ hợp cột tuỳ đợt)
    Tool->>Replica: SELECT ... FROM transactions WHERE <tổ hợp cột không cố định>

    Note over Replica: Khong the index het moi to hop cot co the doi bat ky luc nao
    Note over Replica: Chap nhan Seq Scan / full-scan tren replica

    Replica-->>Tool: Kết quả audit (có thể có độ trễ replication)
    Tool-->>Compliance: Báo cáo audit

    Note over Primary: Trong khi do, Primary van phuc vu insert/update<br/>giao dich thoi gian thuc binh thuong
    Note over Primary: Khong bi anh huong boi tai I/O nang<br/>tu cac query audit full-scan
```
