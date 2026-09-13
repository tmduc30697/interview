# Chỉnh sửa kế hoạch dự án (Gantt chart) trong tool quản lý dự án cộng tác

**Hệ thống:** Một tool quản lý dự án (kiểu Asana/Jira) cho phép nhiều thành viên cùng chỉnh sửa lịch trình task, gán người phụ trách, đổi ngày deadline trên cùng 1 dự án.

**Vai trò của flow:** Vì các thành viên thường sửa các task khác nhau (ít đụng cùng 1 task) nhưng đôi khi 2 người cùng sửa 1 task, flow nên dùng optimistic locking ở cấp độ từng task (không lock cả dự án) để tối đa hóa khả năng làm việc song song.

**Yêu cầu cụ thể:**
- Lock optimistic phải áp dụng ở cấp độ từng task (mỗi task có version riêng), không phải ở cấp độ toàn dự án — để thành viên A sửa task 1 và thành viên B sửa task 2 không bị chặn/xung đột lẫn nhau dù đang cùng mở dự án.
- Mô tả cụ thể: thành viên A đổi deadline task X từ ngày 10 sang ngày 15 (đọc task ở version 3), đồng thời thành viên B gán thêm người phụ trách cho cùng task X (cũng đọc ở version 3) — cả 2 đều sửa các trường khác nhau nhưng cùng version, ai lưu trước sẽ thắng và tăng version, người lưu sau bị từ chối theo optimistic lock thông thường; bàn về việc có nên áp dụng field-level merge (merge các trường không trùng nhau) thay vì từ chối toàn bộ, và ưu/nhược điểm của việc đó.
- Khi task đang được kéo-thả để đổi thứ tự trên Gantt chart (thao tác kéo thả có thể gửi nhiều update nhỏ liên tiếp trong thời gian ngắn), cần quy định rõ có nên coi mỗi lần kéo-thả là 1 version riêng (dễ tạo conflict giả nếu 2 người cùng kéo-thả các task khác nhau ảnh hưởng tới thứ tự chung) hay dùng cơ chế khác (ví dụ optimistic lock ở cấp "toàn bộ thứ tự danh sách task" thay vì từng task riêng khi liên quan tới sắp xếp).
- Nếu task bị xóa bởi 1 thành viên đúng lúc thành viên khác đang sửa task đó (gửi update dựa trên version cũ trước khi bị xóa), request update phải nhận lỗi rõ ràng "task đã bị xóa", không tạo lại task đã xóa hoặc gây lỗi không rõ nguyên nhân.
- Với các trường có tần suất tranh chấp cao hơn (ví dụ % hoàn thành task được nhiều người cùng cập nhật liên tục), đánh giá xem optimistic lock có phù hợp hay nên chuyển sang cơ chế khác (ví dụ increment atomic riêng cho trường đó) để giảm tỷ lệ conflict phải retry liên tục gây trải nghiệm khó chịu.

---

# Base ERD — Quản lý dự án chưa có cơ chế kiểm soát tương tranh

Đây là **base**: trạng thái tool quản lý dự án dạng Gantt chart *trước khi* áp dụng optimistic locking. Suy luận từ đề bài, base đã có `PROJECT` chứa nhiều `TASK`, mỗi task có thể có nhiều người phụ trách (`TASK_ASSIGNEE`), và `PROJECT_MEMBER` xác định ai là thành viên của dự án. Task đã có `deadline`, `percent_complete`, `order_index` để vẽ Gantt chart, nhưng chưa có trường `version` hay bất kỳ cơ chế phát hiện xung đột nào — hai người sửa cùng lúc sẽ ghi đè lẫn nhau (lost update).

```mermaid
erDiagram
    PROJECT ||--o{ TASK : contains
    PROJECT ||--o{ PROJECT_MEMBER : has
    USER ||--o{ PROJECT_MEMBER : "là thành viên"
    TASK ||--o{ TASK_ASSIGNEE : "có người phụ trách"
    USER ||--o{ TASK_ASSIGNEE : "được gán"

    PROJECT {
        string id PK
        string name
    }
    PROJECT_MEMBER {
        string id PK
        string project_id FK
        string user_id FK
        string role
    }
    USER {
        string id PK
        string name
    }
    TASK {
        string id PK
        string project_id FK
        string title
        date start_date
        date deadline
        int percent_complete
        int order_index "vị trí trên Gantt chart"
        datetime updated_at
    }
    TASK_ASSIGNEE {
        string id PK
        string task_id FK
        string user_id FK
    }
```

---

# Base sequence — Update task (không có kiểm soát tương tranh, lost update)

Đây là **base**, flow chỉnh sửa task hiện tại: server ghi đè trực tiếp theo dữ liệu request gửi lên, không kiểm tra task đã bị người khác sửa từ lúc mình đọc hay chưa. Đây chính là tiền đề cho vấn đề nêu ở yêu cầu 1 và 2 của đề bài — thành viên A đổi deadline và thành viên B gán thêm người phụ trách cho cùng task X, ai lưu sau sẽ ghi đè mất thay đổi hợp lệ của người lưu trước.

```mermaid
sequenceDiagram
    actor A as Thành viên A
    actor B as Thành viên B
    participant Server
    participant DB as Database

    A->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (deadline=10, assignee=[])
    Server-->>A: task X

    B->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (deadline=10, assignee=[])
    Server-->>B: task X

    A->>Server: PUT task X (deadline=15)
    Server->>DB: UPDATE task X SET deadline=15
    DB-->>Server: OK
    Server-->>A: Lưu thành công

    B->>Server: PUT task X (assignee=[U2]) dựa trên dữ liệu cũ đã đọc
    Note over Server,DB: Server không kiểm tra task đã đổi kể từ lúc B đọc
    Server->>DB: UPDATE task X SET assignee=[U2], deadline=10 (ghi đè theo payload cũ của B)
    DB-->>Server: OK
    Server-->>B: Lưu thành công

    Note over DB: Deadline=15 của A bị ghi đè về 10, lost update không ai biết
```

---

# Base sequence — Reorder tasks drag-drop (mỗi lần kéo thả ghi đè trực tiếp order_index)

Đây là **base**, flow kéo thả sắp xếp lại task trên Gantt chart: mỗi lần kéo thả gửi thẳng `order_index` mới xuống DB, không có version hay coalescing gì cả. Đây là tiền đề cho yêu cầu 3 của đề bài — khi 2 thành viên cùng kéo thả các task khác nhau nhưng cùng ảnh hưởng tới thứ tự chung, việc ghi đè trực tiếp liên tục dễ làm thứ tự cuối cùng bị sai lệch ngoài ý muốn.

```mermaid
sequenceDiagram
    actor C as Thành viên C
    actor D as Thành viên D
    participant Server
    participant DB as Database

    Note over C,D: Cả 2 đang mở cùng Gantt chart, thứ tự hiện tại [T1, T2, T3]

    C->>Server: Kéo T3 lên trước T1 (order_index: T3=1, T1=2, T2=3)
    Server->>DB: UPDATE order_index cho T3, T1, T2
    DB-->>Server: OK

    D->>Server: Kéo T2 lên đầu, dựa trên thứ tự cũ D thấy [T1, T2, T3]
    Note over Server,DB: Server không biết C vừa đổi thứ tự, không kiểm tra gì thêm
    Server->>DB: UPDATE order_index cho T2=1, T1=2, T3=3

    DB-->>Server: OK
    Note over DB: Kết quả cuối [T2, T1, T3] - thay đổi vị trí T3 lên đầu của C bị mất hoàn toàn, không ai được báo conflict
```

---

# Enhance ERD — Optimistic locking cấp task, order riêng cho danh sách, soft delete

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm các trường/entity sau, ứng trực tiếp với từng yêu cầu:

- `TASK.version` — đáp ứng yêu cầu 1 (optimistic lock ở cấp độ từng task, không phải cấp dự án).
- `TASK.deleted_at` (soft delete thay vì xóa cứng) — đáp ứng yêu cầu 4 (phát hiện update lên task đã bị xóa để trả lỗi rõ ràng thay vì tạo lại hoặc lỗi mơ hồ).
- Entity mới `PROJECT_TASK_ORDER` với `order_version` riêng — đáp ứng yêu cầu 3 (tách version của "thứ tự toàn bộ danh sách" ra khỏi version của từng task, để kéo-thả không làm tăng version của task và không gây conflict giả khi 2 người kéo-thả các task khác nhau).
- `TASK.percent_complete` giữ nguyên kiểu, nhưng được cập nhật qua atomic increment ở DB thay vì qua optimistic lock — đáp ứng yêu cầu 5 (giảm tỷ lệ conflict phải retry cho trường bị tranh chấp cao). Không cần thêm cột mới, chỉ đổi cách ghi (xem sequence diagram tương ứng).

```mermaid
erDiagram
    PROJECT ||--o{ TASK : contains
    PROJECT ||--o{ PROJECT_MEMBER : has
    USER ||--o{ PROJECT_MEMBER : "là thành viên"
    TASK ||--o{ TASK_ASSIGNEE : "có người phụ trách"
    USER ||--o{ TASK_ASSIGNEE : "được gán"
    PROJECT ||--|| PROJECT_TASK_ORDER : "có thứ tự task"

    PROJECT {
        string id PK
        string name
    }
    PROJECT_MEMBER {
        string id PK
        string project_id FK
        string user_id FK
        string role
    }
    USER {
        string id PK
        string name
    }
    TASK {
        string id PK
        string project_id FK
        string title
        date start_date
        date deadline
        int percent_complete "cập nhật bằng atomic increment, không qua version check"
        int version "optimistic lock riêng cho từng task"
        datetime deleted_at "soft delete, null nếu chưa xóa"
        datetime updated_at
    }
    TASK_ASSIGNEE {
        string id PK
        string task_id FK
        string user_id FK
    }
    PROJECT_TASK_ORDER {
        string project_id PK
        string ordered_task_ids "JSON array thứ tự task hiện tại"
        int order_version "version riêng cho toàn bộ thứ tự danh sách, tách khỏi TASK.version"
    }
```

---

# Enhance sequence — Update task (optimistic lock cấp task + field-level merge)

Đây là **enhance** của flow `update-task` đã có ở base. So với base (ghi đè trực tiếp, lost update), nay mỗi update phải kèm `version` đã đọc; nếu version không khớp, hệ thống kiểm tra field-level thay vì từ chối toàn bộ ngay — nếu 2 request sửa các trường không giao nhau (A đổi deadline, B thêm assignee) thì merge cả hai vào bản ghi mới. Đáp ứng đúng kịch bản mô tả ở yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    actor A as Thành viên A
    actor B as Thành viên B
    participant Server
    participant DB as Database

    A->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (version=3, deadline=10, assignee=[])
    Server-->>A: task X (version=3)

    B->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (version=3, deadline=10, assignee=[])
    Server-->>B: task X (version=3)

    A->>Server: PUT task X (deadline=15, version=3)
    Server->>DB: UPDATE task X SET deadline=15, version=4 WHERE version=3
    DB-->>Server: 1 row affected, OK
    Server-->>A: Lưu thành công, version=4

    B->>Server: PUT task X (assignee=[U2], version=3)
    Server->>DB: UPDATE task X SET assignee=[U2], version=4 WHERE version=3
    DB-->>Server: 0 row affected, version hiện tại đã là 4
    Note over Server: Version mismatch, kiểm tra field-level thay vì reject ngay
    Server->>DB: Diff các trường B đổi (assignee) so với các trường A đã đổi (deadline)
    Note over Server: Không giao nhau, field-level merge chấp nhận được
    Server->>DB: UPDATE task X SET assignee=[U2], version=5 WHERE version=4
    DB-->>Server: OK
    Server-->>B: Lưu thành công (đã merge), version=5

    Note over Server,DB: Trade-off, field-level merge tăng khả năng song song nhưng rủi ro nếu 2 trường có phụ thuộc chéo với nhau, ví dụ 1 field validate dựa trên field kia, nên chỉ áp dụng merge cho các trường thực sự độc lập, còn lại vẫn reject theo optimistic lock thông thường
```

---

# Enhance sequence — Reorder tasks drag-drop (version riêng cho toàn bộ thứ tự danh sách)

Đây là **enhance** của flow `reorder-tasks-drag-drop` đã có ở base. So với base (mỗi lần kéo thả ghi đè `order_index` trực tiếp, không kiểm tra gì), nay toàn bộ thao tác kéo thả trong 1 phiên được coalesce thành 1 request cập nhật `ordered_task_ids` duy nhất, kiểm tra bằng `order_version` riêng biệt với `TASK.version` — không dùng optimistic lock ở cấp từng task cho việc sắp xếp. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor C as Thành viên C
    actor D as Thành viên D
    participant Client as Client (debounce kéo thả)
    participant Server
    participant DB as Database

    Note over C,D: order_version hiện tại = 7, thứ tự [T1, T2, T3]

    C->>Client: Kéo T3 lên trước T1 (nhiều sự kiện drag liên tiếp)
    Client->>Client: Coalesce các sự kiện drag thành 1 thứ tự cuối cùng
    Client->>Server: PUT project order (order=[T3,T1,T2], order_version=7)
    Server->>DB: UPDATE PROJECT_TASK_ORDER SET ordered_task_ids=[T3,T1,T2], order_version=8 WHERE order_version=7
    DB-->>Server: 1 row affected, OK
    Server-->>Client: Lưu thành công, order_version=8

    D->>Client: Kéo T2 lên đầu, dựa trên order_version=7 D đã tải trước đó
    Client->>Server: PUT project order (order=[T2,T1,T3], order_version=7)
    Server->>DB: UPDATE PROJECT_TASK_ORDER ... WHERE order_version=7
    DB-->>Server: 0 row affected, order_version hiện tại đã là 8
    Server-->>Client: Conflict, trả về thứ tự mới nhất [T3,T1,T2] và order_version=8
    Client-->>D: Hiển thị "thứ tự đã được cập nhật bởi người khác", D kéo lại trên thứ tự mới

    Note over Server,DB: Vì order_version tách riêng khỏi TASK.version, việc kéo-thả không làm tăng version của từng task, tránh conflict giả khi A/B ở flow update-task đang sửa field khác của task
```

---

# Enhance sequence — Update task đã bị xóa (conflict rõ ràng, không tạo lại nhầm)

Đây là **enhance**, flow mới phát sinh từ đề bài, chưa có ở base (base chưa hỗ trợ xóa task nên chưa có kịch bản này). Khi thành viên E xóa task X (soft delete) đúng lúc thành viên F đang sửa task X dựa trên version cũ trước khi bị xóa, request update phải nhận lỗi rõ ràng "task đã bị xóa" thay vì tạo lại task hoặc báo lỗi version mismatch mơ hồ. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor E as Thành viên E
    actor F as Thành viên F
    participant Server
    participant DB as Database

    F->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (version=5, deleted_at=null)
    Server-->>F: task X (version=5)

    E->>Server: DELETE task X (version=5)
    Server->>DB: UPDATE task X SET deleted_at=now(), version=6 WHERE version=5
    DB-->>Server: OK
    Server-->>E: Xóa thành công

    F->>Server: PUT task X (deadline=20, version=5)
    Server->>DB: SELECT task X WHERE id=X
    DB-->>Server: task X (version=6, deleted_at=<timestamp>)
    Note over Server: Version không khớp (5 vs 6), kiểm tra nguyên nhân trước khi merge
    Server->>DB: Kiểm tra deleted_at, phát hiện task đã bị xóa
    Server-->>F: Lỗi rõ ràng "Task đã bị xóa bởi thành viên khác", không tạo lại task, không áp update
```

---

# Enhance sequence — Update percent_complete (atomic increment, bỏ qua optimistic lock)

Đây là **enhance**, flow mới phát sinh từ đề bài cho riêng trường `percent_complete` — trường có tần suất tranh chấp cao hơn hẳn các trường khác vì nhiều thành viên cùng cập nhật liên tục. Thay vì dùng optimistic lock như flow `update-task` (sẽ gây tỷ lệ conflict/retry rất cao cho trường này), hệ thống dùng atomic increment ở tầng DB, bỏ qua hoàn toàn kiểm tra version cho riêng trường này. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor G as Thành viên G
    actor H as Thành viên H
    participant Server
    participant DB as Database

    Note over G,H: task X đang có percent_complete=40, version=9 (version này không bị đụng tới bởi 2 update dưới đây)

    G->>Server: PATCH task X percent_complete += 10
    Server->>DB: UPDATE task X SET percent_complete = LEAST(100, percent_complete + 10) WHERE id=X
    DB-->>Server: OK, percent_complete=50

    H->>Server: PATCH task X percent_complete += 5 (gần như đồng thời với G)
    Server->>DB: UPDATE task X SET percent_complete = LEAST(100, percent_complete + 5) WHERE id=X
    DB-->>Server: OK, percent_complete=55

    Server-->>G: Cập nhật thành công, percent_complete hiện tại=55
    Server-->>H: Cập nhật thành công, percent_complete hiện tại=55

    Note over Server,DB: Không cần đọc version trước, không cần retry khi conflict, cả 2 update đều được cộng dồn đúng vì DB tự đảm bảo tính atomic của phép increment, đổi lại tính năng này không hỗ trợ merge phức tạp như flow update-task
```
