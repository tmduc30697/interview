# Cache danh sách lớn offline-first bằng IndexedDB cho SaaS dashboard

**Hệ thống:** SaaS B2B dashboard hiển thị danh sách lớn (hàng chục nghìn bản ghi: task, ticket, contact) cần hoạt động mượt và vẫn xem được dữ liệu gần nhất khi mất mạng tạm thời, dùng IndexedDB làm cache tầng client song song với gọi API.

**Vai trò của flow:** Giữ app "cảm giác nhanh" (đọc từ IndexedDB trước, render ngay) đồng thời đồng bộ đúng khi có mạng trở lại và khi mạng không ổn định.

**Yêu cầu cụ thể:**
- Chiến lược cache: đọc từ IndexedDB hiển thị ngay (stale-while-revalidate), gọi API ngầm để lấy bản mới nhất và cập nhật lại UI + IndexedDB — không được để user chỉnh sửa dựa trên dữ liệu stale mà tưởng là mới nhất, phải có chỉ báo rõ ("đang cập nhật..." / thời điểm sync gần nhất).
- Đổi schema lưu trong IndexedDB giữa các version app (thêm field, đổi cấu trúc object store) — phải có cơ chế migration (`onupgradeneeded`) không làm mất dữ liệu cache cũ hoặc crash app cho user đang chạy version cũ chưa update.
- Ghi nhận thao tác user khi offline (sửa/xóa record) vào một "outbox" trong IndexedDB, replay đúng thứ tự lên server khi có mạng lại; xử lý conflict khi bản ghi đó đã bị người khác sửa trên server trong lúc offline.
- Giới hạn dung lượng IndexedDB (quota per-origin của trình duyệt) — có chiến lược eviction (LRU theo bảng/dự án ít dùng nhất) khi gần chạm quota, tránh lỗi ghi thất bại giữa chừng làm hỏng transaction đang dở.
- Đo lường: tỷ lệ cache hit của IndexedDB so với phải gọi API, và độ trễ giữa "hiển thị từ cache" tới "đồng bộ xong dữ liệu mới nhất", để biết cache có thực sự giúp ích hay đang gây hiển thị sai kéo dài.

---

# Base ERD — SaaS dashboard trước khi có cache IndexedDB offline-first

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống **trước khi** có cache tầng client bằng IndexedDB. Đề bài nói tới dashboard hiển thị danh sách lớn task/ticket/contact, nên base chỉ cần đủ Organization/Project và Record (gộp chung task/ticket/contact) ở phía server, đọc/ghi trực tiếp qua API mỗi lần. Chưa có bất kỳ entity nào phía client (cache, outbox, schema version, quota, metric) — toàn bộ phần đó là enhance.

```mermaid
erDiagram
    PROJECT ||--o{ RECORD : contains
    USER ||--o{ RECORD : "edits"

    USER {
        string id PK
        string display_name
    }
    PROJECT {
        string id PK
        string name
    }
    RECORD {
        string id PK
        string project_id FK
        string type "task | ticket | contact"
        string data
        string updated_by FK
        datetime updated_at
    }
```

---

# Base sequence — Load list (luôn gọi API trực tiếp, không có cache)

Đây là **base**, flow tải danh sách bản ghi (task/ticket/contact) ở trạng thái trước enhance. Mỗi lần vào trang, dashboard gọi thẳng API lấy dữ liệu, chờ phản hồi rồi mới render, không có bất kỳ cache cục bộ nào. Nếu mất mạng, người dùng chỉ thấy lỗi hoặc màn hình trống. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi IndexedDB được đọc trước để hiển thị ngay.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant API as List API
    participant DB as Server DB

    User->>App: Mở trang danh sách task/ticket/contact
    App->>API: GET danh sách theo project
    alt Có mạng, API phản hồi bình thường
        API->>DB: Truy vấn danh sách bản ghi
        DB-->>API: Trả về hàng chục nghìn bản ghi
        API-->>App: Trả kết quả
        App-->>User: Render danh sách sau khi có đủ dữ liệu
    else Mất mạng hoặc API lỗi
        API-->>App: Timeout/lỗi kết nối
        App-->>User: Hiển thị lỗi hoặc màn hình trống, không có dữ liệu nào để xem tạm
    end
```

---

# Base sequence — Edit record (ghi thẳng server, thất bại ngay khi offline)

Đây là **base**, flow chỉnh sửa 1 bản ghi ở trạng thái trước enhance. Mọi thao tác sửa/xoá đều gọi thẳng API, không có bước lưu tạm cục bộ nào. Khi mất mạng, thao tác thất bại ngay lập tức và có thể mất nội dung vừa sửa. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi thao tác offline được ghi vào outbox để replay sau.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant API as Record API
    participant DB as Server DB

    User->>App: Sửa nội dung 1 bản ghi (task/ticket/contact)
    App->>API: PATCH record với nội dung mới
    alt Có mạng
        API->>DB: Cập nhật RECORD, updated_at
        API-->>App: Xác nhận đã lưu
        App-->>User: Hiển thị đã lưu thành công
    else Mất mạng
        API-->>App: Request thất bại (network error)
        App-->>User: Báo lỗi lưu thất bại, không có cơ chế lưu tạm hay tự động thử lại sau
    end
```

---

# Enhance ERD — Cache offline-first bằng IndexedDB

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (chỉ có PROJECT/RECORD ở server), phần enhance thêm 6 nhóm entity mới ở phía client (IndexedDB) và đo lường, ứng trực tiếp với các yêu cầu trong đề bài:

- `CACHED_RECORD` (mới, IndexedDB) — bản snapshot record để hiển thị ngay, kèm `last_synced_at` và `is_stale` để biết đang hiển thị dữ liệu cũ hay đã đồng bộ — yêu cầu 1.
- `SCHEMA_VERSION` (mới, meta trong IndexedDB) — theo dõi version schema hiện tại của database cục bộ, phục vụ `onupgradeneeded` migration — yêu cầu 2.
- `OUTBOX_ENTRY` (mới, IndexedDB) — hàng đợi thao tác sửa/xoá thực hiện lúc offline, có `order_seq` để replay đúng thứ tự — yêu cầu 3.
- `SYNC_CONFLICT` (mới) — ghi nhận khi 1 outbox entry replay lên server nhưng bản ghi đã bị người khác sửa trong lúc offline — yêu cầu 3.
- `STORAGE_QUOTA_STATE` + `EVICTION_LOG` (mới) — theo dõi dung lượng IndexedDB đã dùng so với quota per-origin, và lịch sử eviction theo project ít dùng nhất (LRU) — yêu cầu 4.
- `CACHE_METRIC` (mới) — tỉ lệ cache hit so với phải gọi API, và độ trễ từ lúc hiển thị cache tới lúc đồng bộ xong dữ liệu mới nhất — yêu cầu 5.

```mermaid
erDiagram
    PROJECT ||--o{ RECORD : contains
    USER ||--o{ RECORD : edits
    RECORD ||--o| CACHED_RECORD : "cached as (IndexedDB)"
    PROJECT ||--o{ CACHED_RECORD : "grouped for eviction"
    USER ||--o{ OUTBOX_ENTRY : "queues while offline"
    RECORD ||--o{ OUTBOX_ENTRY : "target of"
    OUTBOX_ENTRY ||--o| SYNC_CONFLICT : "may raise"
    PROJECT ||--o{ EVICTION_LOG : "evicted when quota near full"
    PROJECT ||--o{ CACHE_METRIC : "measured per"

    USER {
        string id PK
        string display_name
    }
    PROJECT {
        string id PK
        string name
    }
    RECORD {
        string id PK
        string project_id FK
        string type "task | ticket | contact"
        string data
        string updated_by FK
        datetime updated_at
    }
    CACHED_RECORD {
        string id PK
        string record_id FK
        string project_id FK
        string data_snapshot
        datetime cached_at
        datetime last_synced_at
        boolean is_stale
        datetime last_accessed_at
    }
    SCHEMA_VERSION {
        int db_version PK
        string migration_notes
        datetime migrated_at
    }
    OUTBOX_ENTRY {
        string id PK
        string record_id FK
        string user_id FK
        string action "update | delete"
        string payload
        int order_seq
        string status "pending | synced | conflict"
        datetime created_at
    }
    SYNC_CONFLICT {
        string id PK
        string outbox_entry_id FK
        string server_version_data
        string local_version_data
        string resolution "keep_server | keep_local | manual_merge"
        datetime detected_at
    }
    STORAGE_QUOTA_STATE {
        string origin PK
        int used_bytes
        int quota_bytes
        string eviction_policy "LRU_by_project"
        datetime checked_at
    }
    EVICTION_LOG {
        string id PK
        string project_id FK
        int freed_bytes
        string reason "quota_near_full"
        datetime evicted_at
    }
    CACHE_METRIC {
        string id PK
        string project_id FK
        date window_date
        int cache_hit_count
        int cache_miss_count
        int avg_stale_to_synced_ms
    }
```

---

# Enhance sequence — Load list (stale-while-revalidate, có chỉ báo trạng thái sync)

Đây là **enhance**, cùng flow "load-list" như ở base nhưng thay đổi chiến lược: đọc từ IndexedDB hiển thị ngay lập tức, đồng thời gọi API ngầm để lấy bản mới nhất rồi cập nhật lại UI + IndexedDB. Vì dữ liệu hiển thị ban đầu có thể là stale, hệ thống phải có chỉ báo rõ ràng ("đang cập nhật..." hoặc thời điểm sync gần nhất) để user không nhầm tưởng đây là dữ liệu mới nhất. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (CACHED_RECORD)
    participant API as List API

    User->>App: Mở trang danh sách task/ticket/contact
    App->>IDB: Đọc CACHED_RECORD hiện có cho project này

    alt Có cache cục bộ
        IDB-->>App: Trả về danh sách đã cache, kèm last_synced_at
        App-->>User: Render ngay danh sách từ cache, hiển thị chỉ báo "đang cập nhật... (đồng bộ lần cuối lúc X)"
        Note over User: Nếu user bấm sửa ngay lúc này, UI phải nhắc rõ dữ liệu có thể chưa phải mới nhất
    else Không có cache (lần đầu mở)
        App-->>User: Hiển thị trạng thái đang tải
    end

    App->>API: Gọi API ngầm lấy dữ liệu mới nhất cho project
    API-->>App: Trả về danh sách mới nhất

    App->>IDB: Cập nhật CACHED_RECORD theo dữ liệu mới, is_stale=false, last_synced_at=now
    App-->>User: Cập nhật lại UI với dữ liệu mới nhất, gỡ chỉ báo "đang cập nhật...", hiển thị thời điểm sync mới
```

---

# Enhance sequence — Migration schema IndexedDB giữa các version app

Đây là **enhance**, flow hoàn toàn mới, xử lý việc đổi schema lưu trong IndexedDB giữa các version app (thêm field, đổi cấu trúc object store). Phải dùng cơ chế `onupgradeneeded` để migrate mà không làm mất dữ liệu cache cũ, và không được crash app cho user đang chạy version cũ chưa kịp cập nhật. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard (version mới)
    participant IDB as IndexedDB
    participant Meta as SCHEMA_VERSION

    User->>App: Mở app sau khi đã cập nhật lên version mới
    App->>IDB: Mở database, yêu cầu db_version mới (ví dụ tăng từ 3 lên 4)

    alt db_version trên máy user cũ hơn version app yêu cầu
        IDB->>App: Kích hoạt sự kiện onupgradeneeded
        App->>IDB: Chạy migration, ví dụ thêm field mới vào object store hoặc tạo index mới, giữ nguyên dữ liệu cũ
        App->>Meta: Cập nhật SCHEMA_VERSION(db_version=4, migration_notes, migrated_at=now)
        IDB-->>App: Mở database thành công với schema mới, dữ liệu CACHED_RECORD cũ vẫn còn nguyên
    else db_version đã khớp
        IDB-->>App: Mở database bình thường, không cần migrate
    end

    App-->>User: Tiếp tục dùng app bình thường với dữ liệu cache cũ đã được migrate

    Note over App: Trường hợp riêng, user khác vẫn đang chạy version app cũ (chưa update)
    App->>IDB: Version app cũ mở database với db_version cũ hơn bản đã được version mới nâng lên
    IDB-->>App: Trình duyệt từ chối mở ở version thấp hơn version hiện có của database
    App-->>User: App version cũ phát hiện không mở được cache, tự chuyển sang gọi thẳng API thay vì crash
```

---

# Enhance sequence — Edit record (ghi vào outbox khi offline thay vì thất bại ngay)

Đây là **enhance**, cùng flow "edit-record" như ở base nhưng thay đổi hành vi khi mất mạng: thay vì thất bại ngay, thao tác sửa/xoá được ghi vào outbox trong IndexedDB kèm thứ tự thực hiện, để replay lên server khi có mạng lại (chi tiết replay và xử lý conflict ở flow outbox-replay-conflict riêng). Đáp ứng phần đầu của yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant API as Record API
    participant IDB as IndexedDB (CACHED_RECORD, OUTBOX_ENTRY)

    User->>App: Sửa nội dung 1 bản ghi (task/ticket/contact)
    App->>IDB: Cập nhật CACHED_RECORD cục bộ ngay để UI phản hồi tức thì

    alt Có mạng
        App->>API: PATCH record với nội dung mới
        API-->>App: Xác nhận đã lưu trên server
        App->>IDB: Cập nhật CACHED_RECORD.last_synced_at=now, is_stale=false
        App-->>User: Hiển thị đã lưu thành công
    else Mất mạng
        App->>IDB: Ghi OUTBOX_ENTRY(record_id, action=update, payload, order_seq tăng dần, status=pending)
        App-->>User: Hiển thị đã lưu tạm cục bộ, sẽ đồng bộ khi có mạng lại
        Note over IDB: Nội dung sửa không bị mất như ở base, đã nằm an toàn trong outbox chờ replay
    end
```

---

# Enhance sequence — Replay outbox khi có mạng lại, xử lý conflict

Đây là **enhance**, flow hoàn toàn mới, tiếp nối flow "edit-record" khi mạng có lại. Các thao tác offline đã ghi trong outbox phải được replay đúng thứ tự (`order_seq`) lên server, và phải xử lý rõ trường hợp bản ghi đó đã bị người khác sửa trên server trong lúc offline. Đáp ứng phần sau của yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (OUTBOX_ENTRY)
    participant API as Record API
    participant DB as Server DB
    participant Conflict as SYNC_CONFLICT

    Note over App: Trình duyệt phát sự kiện online trở lại
    App->>IDB: Lấy toàn bộ OUTBOX_ENTRY status=pending, sắp theo order_seq tăng dần

    loop Với từng outbox entry theo đúng thứ tự
        App->>API: Gửi thao tác (update/delete) kèm version/updated_at mà local biết lúc bắt đầu sửa
        API->>DB: Kiểm tra RECORD.updated_at hiện tại trên server

        alt Không ai sửa bản ghi này trong lúc offline
            DB-->>API: Áp dụng thay đổi bình thường
            API-->>App: Xác nhận replay thành công
            App->>IDB: Đánh dấu OUTBOX_ENTRY.status=synced
        else Bản ghi đã bị người khác sửa trên server trong lúc offline
            DB-->>API: Phát hiện updated_at trên server mới hơn baseline local
            API-->>App: Trả về conflict, kèm bản ghi hiện tại trên server
            App->>Conflict: Ghi SYNC_CONFLICT(server_version_data, local_version_data)
            App->>IDB: Đánh dấu OUTBOX_ENTRY.status=conflict
            App-->>User: Hiển thị rõ xung đột, cho user chọn giữ bản của mình, giữ bản server, hay merge thủ công
        end
    end
```

---

# Enhance sequence — Eviction khi gần chạm quota IndexedDB

Đây là **enhance**, flow hoàn toàn mới, xử lý giới hạn dung lượng IndexedDB theo quota per-origin của trình duyệt. Khi gần chạm quota, hệ thống phải áp dụng chiến lược eviction LRU theo project ít dùng nhất, và tránh để lỗi ghi thất bại giữa chừng làm hỏng transaction đang dở. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (CACHED_RECORD)
    participant Quota as STORAGE_QUOTA_STATE
    participant Evict as EVICTION_LOG

    App->>Quota: Kiểm tra dung lượng đã dùng so với quota per-origin trước khi ghi cache mới
    alt Còn nhiều dung lượng trống
        App->>IDB: Ghi CACHED_RECORD bình thường trong 1 transaction
        IDB-->>App: Ghi thành công
    else Gần chạm quota (ví dụ trên 90%)
        App->>Quota: Xác định các project có last_accessed_at cũ nhất (LRU)
        Quota-->>App: Trả về danh sách project ít dùng nhất cần giải phóng

        App->>IDB: Mở transaction riêng để xoá CACHED_RECORD của các project đó, tách biệt với transaction ghi dữ liệu mới
        IDB-->>App: Xoá thành công, giải phóng dung lượng
        App->>Evict: Ghi EVICTION_LOG(project_id, freed_bytes, reason=quota_near_full)

        App->>IDB: Sau khi đã giải phóng đủ, mới mở transaction mới để ghi CACHED_RECORD cần lưu
        alt Ghi thành công
            IDB-->>App: OK
        else Vẫn thất bại do quota (trường hợp hiếm)
            IDB-->>App: Lỗi ghi, transaction tự rollback nguyên vẹn nhờ tách transaction riêng từ bước xoá
            App-->>User: Bỏ qua cache cho lần này, vẫn hiển thị dữ liệu vừa gọi API, không để transaction dở dang làm hỏng cache hiện có
        end
    end
```

---

# Enhance sequence — Đo cache hit rate và độ trễ đồng bộ

Đây là **enhance**, flow hoàn toàn mới, đáp ứng yêu cầu 5 của đề bài: đo tỉ lệ cache hit của IndexedDB so với phải gọi API, và độ trễ giữa lúc hiển thị từ cache tới lúc đồng bộ xong dữ liệu mới nhất, để biết cache có thực sự giúp ích hay đang gây hiển thị sai kéo dài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant IDB as IndexedDB (CACHED_RECORD)
    participant API as List API
    participant Metric as CACHE_METRIC
    actor Analyst as Kỹ sư / Product Analyst

    User->>App: Mở trang danh sách
    App->>IDB: Kiểm tra có CACHED_RECORD cho project này không

    alt Có cache, dùng được ngay (cache hit)
        IDB-->>App: Trả về dữ liệu cache, ghi lại thời điểm hiển thị cache_shown_at
        App->>Metric: Tăng cache_hit_count cho project
    else Không có cache, phải chờ API (cache miss)
        App->>Metric: Tăng cache_miss_count cho project
    end

    App->>API: Gọi API lấy dữ liệu mới nhất (ngầm nếu cache hit, đồng bộ nếu cache miss)
    API-->>App: Trả về dữ liệu mới nhất tại thời điểm synced_at

    alt Trước đó là cache hit
        App->>Metric: Tính stale_to_synced_ms = synced_at trừ cache_shown_at, cộng dồn vào avg_stale_to_synced_ms
    end

    loop Định kỳ
        Metric-->>Analyst: Báo cáo cache_hit_count, cache_miss_count, avg_stale_to_synced_ms theo từng project
        Analyst->>Analyst: Đánh giá, nếu avg_stale_to_synced_ms quá lớn thì cache đang hiển thị sai kéo dài, cần xem lại tần suất revalidate
    end
```
