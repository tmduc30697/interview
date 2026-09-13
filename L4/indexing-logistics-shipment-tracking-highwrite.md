# Logistics tra cứu và theo dõi vận đơn với tần suất ghi trạng thái cao

**Hệ thống:** Nền tảng logistics quản lý vận đơn (shipment), mỗi vận đơn được cập nhật trạng thái liên tục nhiều lần trong ngày (pickup, in-transit, out-for-delivery, delivered...) qua nhiều trạm trung chuyển khác nhau.

**Vai trò của flow:** Cho phép khách hàng và bộ phận vận hành tra cứu nhanh theo mã vận đơn, theo trạng thái hiện tại, hoặc theo khoảng thời gian, trong khi bảng dữ liệu liên tục bị ghi cập nhật với tần suất cao.

**Yêu cầu cụ thể:**
- Unique index trên `tracking_code` phải đảm bảo khách hàng tra cứu tức thời, nhưng nếu mã vận đơn được sinh ngẫu nhiên không tuần tự, việc insert liên tục vận đơn mới có thể gây phân mảnh index nhanh hơn — cần cân nhắc cách sinh mã vừa đảm bảo tính duy nhất/khó đoán, vừa thân thiện với cấu trúc index.
- Dashboard vận hành cần lọc vận đơn theo trạng thái hiện tại, nhưng trạng thái đổi liên tục nghĩa là mỗi lần update phải xoá và chèn lại entry trong index theo status — với hàng triệu vận đơn đang active, chi phí ghi này rất lớn; cần cân nhắc partial index chỉ bao phủ các trạng thái "đang xử lý" thay vì index toàn bộ trạng thái kể cả các đơn đã delivered từ lâu.
- Lịch sử thay đổi trạng thái (nhiều dòng cho mỗi vận đơn) cần composite index (shipment_id, changed_at) để trả đúng thứ tự timeline cho khách xem, nhưng bảng lịch sử tăng vô hạn theo thời gian — cần có chiến lược archive/partition dữ liệu cũ mà không phá vỡ hiệu năng của index đang phục vụ dữ liệu gần đây.
- Hệ thống cần chạy định kỳ (vd mỗi phút) một truy vấn tìm các vận đơn trễ SLA (estimated_delivery đã qua nhưng status khác delivered) trên bảng có quy mô lớn — quét toàn bảng mỗi phút là không khả thi, cần thiết kế index (partial/composite) riêng tối ưu cho đúng dạng truy vấn cảnh báo này.
- Nhiều trạm trung chuyển ghi cập nhật trạng thái đồng thời cho các vận đơn khác nhau — cần chọn cấu trúc index/storage tránh việc nhiều ghi đồng thời dồn vào cùng vài trang index "nóng" (hot page), gây lock contention làm chậm toàn bộ hệ thống dù các vận đơn được cập nhật không liên quan tới nhau.

---

# Base ERD - Quản lý vận đơn cơ bản

Đây là **base**: trạng thái dữ liệu trước khi áp đề bài tối ưu index. Hệ thống đã có bảng `shipments` với unique index trên `tracking_code` để tra cứu tức thời, và bảng `shipment_status_history` lưu lịch sử thay đổi trạng thái, nhưng **chưa có** partial index cho status, chưa có composite index (shipment_id, changed_at) cho lịch sử, và chưa có index riêng phục vụ truy vấn cảnh báo SLA - đây chính là những khoảng trống mà enhance sẽ lấp vào.

```mermaid
erDiagram
    SHIPMENT {
        bigint id PK
        string tracking_code UK "unique btree index idx_shipment_tracking_code, dung tra cuu khach hang"
        string status "khong co index rieng, filter theo status phai quet toan bang"
        string origin
        string destination
        timestamp estimated_delivery "khong co index, chua ho tro truy van SLA hieu qua"
        timestamp created_at
        timestamp updated_at
    }
    SHIPMENT_STATUS_HISTORY {
        bigint id PK
        bigint shipment_id FK "chi co FK, khong co index composite theo (shipment_id, changed_at)"
        string status
        string station_code "ma tram trung chuyen thuc hien cap nhat"
        timestamp changed_at
        string note
    }
    SHIPMENT ||--o{ SHIPMENT_STATUS_HISTORY : "co nhieu dong lich su trang thai"
```

---

# Base - Flow trạm cập nhật trạng thái vận đơn

Đây là **base**, sequence diagram cho flow **update-shipment-status**: một trạm trung chuyển ghi nhận thay đổi trạng thái cho vận đơn. Flow này được chọn vì đây chính là nguồn ghi tần suất cao mà đề bài tập trung tối ưu index xung quanh nó (partial index status, composite index history, tránh hot page khi nhiều trạm ghi đồng thời).

```mermaid
sequenceDiagram
    participant StationA as Trạm A
    participant StationB as Trạm B
    participant API as Shipment Service
    participant DB as PostgreSQL

    par Trạm A cập nhật vận đơn X
        StationA->>API: PATCH /shipments/{trackingCodeX}/status (out_for_delivery)
        API->>DB: UPDATE shipments SET status, updated_at WHERE tracking_code = X
        Note over DB: WHERE dùng unique index tracking_code để seek, không có index riêng theo status
        API->>DB: INSERT INTO shipment_status_history (shipment_id=X, status, station_code=A, changed_at)
        Note over DB: bảng history chỉ có PK tự tăng, insert rẻ nhưng chưa tối ưu cho truy vấn timeline sau này
    and Trạm B cập nhật vận đơn Y cùng lúc
        StationB->>API: PATCH /shipments/{trackingCodeY}/status (delivered)
        API->>DB: UPDATE shipments SET status, updated_at WHERE tracking_code = Y
        API->>DB: INSERT INTO shipment_status_history (shipment_id=Y, status, station_code=B, changed_at)
    end
    DB-->>API: OK cho cả 2 giao dịch
    API-->>StationA: 200 OK
    API-->>StationB: 200 OK
    Note over DB: chưa có cấu trúc index nào được thiết kế riêng để phân tán ghi đồng thời từ nhiều trạm, tiềm ẩn rủi ro tranh chấp trang khi tải tăng
```

---

# Base - Flow khách tra cứu theo mã vận đơn

Đây là **base**, sequence diagram cho flow **lookup-by-tracking-code**: khách hàng tra cứu vận đơn bằng mã tracking_code. Flow này được chọn vì nó là lý do unique index trên `tracking_code` tồn tại ngay từ base, và là flow phải luôn giữ nhanh dù bảng bị ghi liên tục - chính là mối lo mà đề bài nêu ở yêu cầu sinh mã tracking_code thân thiện với index.

```mermaid
sequenceDiagram
    participant Customer as Khách hàng
    participant API as Shipment Service
    participant DB as PostgreSQL

    Customer->>API: GET /shipments/{tracking_code}
    API->>DB: SELECT * FROM shipments WHERE tracking_code = ?
    Note over DB: dùng unique index idx_shipment_tracking_code, hiện tại còn nhỏ nên tra cứu nhanh
    DB-->>API: thông tin vận đơn
    API-->>Customer: 200 OK

    Note over DB: mã tracking_code hiện sinh ngẫu nhiên không tuần tự, mỗi insert vận đơn mới rơi vào vị trí bất kỳ trong B-tree, càng nhiều insert index càng dễ phân mảnh, về lâu dài có thể làm chậm dần chính flow tra cứu này
```

---

# Enhance ERD - Tối ưu index cho vận đơn tần suất ghi cao

Đây là **enhance**: cùng 2 bảng `shipments` và `shipment_status_history` như base, nhưng bổ sung: partial index cho status "đang xử lý" (thay vì index toàn bộ trạng thái), composite index (shipment_id, changed_at) cho lịch sử, index riêng phục vụ truy vấn cảnh báo SLA, chiến lược tracking_code thân thiện với index khi insert, và tách partition nóng/lạnh cho lịch sử để archive dữ liệu cũ mà không ảnh hưởng index đang phục vụ dữ liệu gần đây.

```mermaid
erDiagram
    SHIPMENT {
        bigint id PK
        string tracking_code UK "unique index idx_shipment_tracking_code, sinh dang K-sortable theo time_bucket cong random suffix de giam phan manh khi insert"
        string status "partial index idx_shipment_active_status (status) WHERE status khac delivered va cancelled, phuc vu dashboard loc don dang xu ly"
        string origin
        string destination
        timestamp estimated_delivery "partial index idx_sla_alert (estimated_delivery) WHERE status khac delivered, phuc vu job canh bao SLA dinh ky"
        timestamp created_at
        timestamp updated_at
    }
    SHIPMENT_STATUS_HISTORY {
        bigint id PK
        bigint shipment_id FK "composite index idx_history_shipment_changed (shipment_id, changed_at)"
        string status
        string station_code
        timestamp changed_at "partition theo khoang thoi gian, giu partition gan day nho gon va nong"
        string note
    }
    SHIPMENT_STATUS_HISTORY_ARCHIVE {
        bigint id PK
        bigint shipment_id "cung mang shipment_id nhu partition nong, giu index (shipment_id, changed_at) rieng tren du lieu cu"
        string status
        string station_code
        timestamp changed_at
        string note
    }
    SHIPMENT ||--o{ SHIPMENT_STATUS_HISTORY : "lich su gan day, partition nong con index nho"
    SHIPMENT ||--o{ SHIPMENT_STATUS_HISTORY_ARCHIVE : "lich su cu da archive, partition lanh tach rieng"
```

---

# Enhance - Flow trạm cập nhật trạng thái vận đơn

Đây là **enhance** của flow **update-shipment-status**. So với base, flow này thay đổi ở 3 điểm: (1) UPDATE status giờ tương tác với partial index idx_shipment_active_status - entry chỉ tồn tại khi status còn "đang xử lý" và tự động bị xoá khỏi index khi chuyển sang delivered, (2) INSERT lịch sử ghi vào composite index (shipment_id, changed_at) thay vì bảng không có index chuyên biệt, (3) vì shipment_id là cột dẫn đầu của composite index, các trạm khác nhau ghi vào các vị trí khác nhau trong B-tree, giảm tranh chấp trang nóng so với base.

```mermaid
sequenceDiagram
    participant StationA as Trạm A
    participant StationB as Trạm B
    participant API as Shipment Service
    participant DB as PostgreSQL

    par Trạm A cập nhật vận đơn X sang out_for_delivery
        StationA->>API: PATCH /shipments/{trackingCodeX}/status (out_for_delivery)
        API->>DB: UPDATE shipments SET status='out_for_delivery' WHERE tracking_code = X
        Note over DB: status cũ và mới đều thoả điều kiện WHERE của idx_shipment_active_status, chỉ cập nhật giá trị trong index, không thêm/xoá entry
        API->>DB: INSERT INTO shipment_status_history (shipment_id=X, status, station_code=A, changed_at=now())
        Note over DB: ghi vào composite index idx_history_shipment_changed (shipment_id, changed_at), vị trí ghi phụ thuộc shipment_id nên phân tán trong B-tree
    and Trạm B cập nhật vận đơn Y sang delivered
        StationB->>API: PATCH /shipments/{trackingCodeY}/status (delivered)
        API->>DB: UPDATE shipments SET status='delivered' WHERE tracking_code = Y
        Note over DB: status mới delivered không còn thoả điều kiện partial index, entry của Y bị xoá khỏi idx_shipment_active_status, index tự động nhỏ lại theo thời gian
        API->>DB: INSERT INTO shipment_status_history (shipment_id=Y, status, station_code=B, changed_at=now())
    end
    DB-->>API: OK cho cả 2 giao dịch
    API-->>StationA: 200 OK
    API-->>StationB: 200 OK
    Note over DB: shipment_id ngẫu nhiên giữa các trạm giúp ghi đồng thời rải đều trên nhiều trang của composite index, thay vì dồn vào một trang cuối như khi chỉ index theo changed_at một mình
```

---

# Enhance - Flow tạo vận đơn mới với tracking_code thân thiện index

Đây là **enhance**, sequence diagram cho flow mới **create-shipment**: tạo vận đơn và sinh mã tracking_code. Base đã có unique index trên tracking_code nhưng không mô tả cách sinh mã; flow này được thêm để làm rõ chiến lược sinh mã mới - vừa khó đoán vừa hạn chế phân mảnh index khi insert liên tục với tần suất cao, đúng yêu cầu đầu tiên của đề bài.

```mermaid
sequenceDiagram
    participant Client as Merchant/App tạo đơn
    participant API as Shipment Service
    participant DB as PostgreSQL

    Client->>API: POST /shipments (origin, destination, estimated_delivery)
    API->>API: Sinh tracking_code = time_bucket hiện tại (vd theo giờ) + chuỗi random 6 ký tự
    Note over API: time_bucket làm mã tăng dần thô theo thời gian nhưng vẫn đủ random trong từng bucket để khó đoán, khác với UUIDv4 hoàn toàn ngẫu nhiên
    API->>DB: INSERT INTO shipments (tracking_code, status='created', origin, destination, estimated_delivery)
    Note over DB: giá trị tracking_code mới rơi vào vùng gần cuối B-tree của unique index trong một khoảng thời gian ngắn, giảm page-split/phân mảnh so với insert hoàn toàn ngẫu nhiên trên toàn bộ không gian khoá
    Note over DB: trong từng time_bucket mã vẫn đủ random để khách hàng không đoán được mã liền kề của vận đơn khác
    DB-->>API: shipment_id, tracking_code
    API-->>Client: 201 Created (tracking_code)
```

---

# Enhance - Flow dashboard vận hành lọc vận đơn theo trạng thái

Đây là **enhance**, sequence diagram cho flow mới **filter-active-shipments-dashboard**: bộ phận vận hành lọc danh sách vận đơn đang xử lý. Flow này hoàn toàn mới so với base (base chưa có index nào phục vụ lọc theo status), được thêm để minh hoạ trực tiếp partial index idx_shipment_active_status giải quyết yêu cầu thứ hai của đề bài.

```mermaid
sequenceDiagram
    participant Operator as Nhân viên vận hành
    participant Dashboard as Ops Dashboard
    participant API as Shipment Service
    participant DB as PostgreSQL

    Operator->>Dashboard: Lọc danh sách vận đơn đang xử lý
    Dashboard->>API: GET /shipments?status=active
    API->>DB: SELECT * FROM shipments WHERE status NOT IN (delivered, cancelled) ORDER BY updated_at DESC
    Note over DB: query planner dùng partial index idx_shipment_active_status, index này chỉ chứa các dòng đang active nên kích thước nhỏ ổn định dù bảng shipments có hàng triệu đơn đã delivered từ lâu
    DB-->>API: danh sách vận đơn active
    API-->>Dashboard: 200 OK (list)
    Dashboard-->>Operator: hiển thị bảng lọc theo trạng thái
```

---

# Enhance - Flow khách xem timeline lịch sử vận đơn

Đây là **enhance**, sequence diagram cho flow mới **view-shipment-timeline**: khách hàng xem toàn bộ lịch sử thay đổi trạng thái theo đúng thứ tự thời gian. Flow này hoàn toàn mới so với base (base chưa có index hỗ trợ truy vấn timeline hiệu quả), được thêm để minh hoạ composite index (shipment_id, changed_at) và chiến lược partition nóng/lạnh giải quyết yêu cầu thứ ba của đề bài.

```mermaid
sequenceDiagram
    participant Customer as Khách hàng
    participant API as Shipment Service
    participant DB as PostgreSQL partitioned history

    Customer->>API: GET /shipments/{tracking_code}/timeline
    API->>DB: SELECT id FROM shipments WHERE tracking_code = ?
    DB-->>API: shipment_id
    API->>DB: SELECT status, station_code, changed_at FROM shipment_status_history WHERE shipment_id = ? ORDER BY changed_at
    Note over DB: bảng shipment_status_history được partition theo changed_at (vd theo tháng), planner chỉ quét các partition liên quan thay vì toàn bảng
    Note over DB: composite index idx_history_shipment_changed (shipment_id, changed_at) trên từng partition trả đúng thứ tự timeline mà không cần sort riêng
    DB-->>API: danh sách các dòng trạng thái theo thứ tự thời gian
    API-->>Customer: 200 OK (timeline)

    Note over DB: partition cũ định kỳ được chuyển sang SHIPMENT_STATUS_HISTORY_ARCHIVE, giữ partition nóng phục vụ dữ liệu gần đây luôn nhỏ gọn và index không bị phình theo thời gian
```

---

# Enhance - Flow job định kỳ tìm vận đơn trễ SLA

Đây là **enhance**, sequence diagram cho flow mới **sla-alert-job**: job chạy định kỳ mỗi phút để tìm các vận đơn đã quá estimated_delivery nhưng chưa delivered. Flow này hoàn toàn mới so với base (base chưa có index nào phục vụ dạng truy vấn này), được thêm để minh hoạ index riêng cho cảnh báo SLA giải quyết yêu cầu thứ tư của đề bài.

```mermaid
sequenceDiagram
    participant Scheduler as Cron mỗi phút
    participant Worker as SLA Alert Worker
    participant DB as PostgreSQL
    participant Notifier as Hệ thống cảnh báo

    Scheduler->>Worker: Kích hoạt job kiểm tra SLA
    Worker->>DB: SELECT tracking_code, estimated_delivery, status FROM shipments WHERE status khac delivered AND estimated_delivery < now()
    Note over DB: planner dùng index riêng idx_sla_alert, partial WHERE status khac delivered kết hợp composite trên estimated_delivery, chỉ quét tập con đơn chưa giao thay vì full table scan hàng triệu dòng mỗi phút
    DB-->>Worker: danh sách vận đơn trễ SLA
    Worker->>Notifier: Gửi cảnh báo cho từng vận đơn trễ
    Notifier-->>Worker: đã gửi
    Worker-->>Scheduler: Hoàn tất job
```
