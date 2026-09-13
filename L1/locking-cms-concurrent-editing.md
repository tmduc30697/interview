# Chỉnh sửa bài viết CMS bởi nhiều biên tập viên

**Hệ thống:** Một CMS quản lý nội dung cho tòa soạn báo online, nhiều biên tập viên có thể cùng mở và sửa 1 bài viết (ít xảy ra nhưng không hiếm, ví dụ 2 biên tập viên cùng ca trực).

**Vai trò của flow:** Vì xung đột sửa đồng thời hiếm xảy ra nhưng cần trải nghiệm mượt (không muốn khóa bài viết chặn người khác chỉ vì 1 người đang mở xem), flow lưu bài viết nên dùng optimistic locking dựa trên version, phát hiện xung đột tại thời điểm lưu thay vì chặn ngay từ khi mở bài.

**Yêu cầu cụ thể:**
- Mỗi bài viết có cột `version` tăng dần mỗi lần lưu; khi biên tập viên A mở bài ở version 5, sửa nội dung rồi lưu, request lưu phải gửi kèm version 5 đã đọc và server chỉ chấp nhận nếu version hiện tại trong DB vẫn là 5 (update kèm điều kiện `WHERE version = 5`), nếu không khớp phải trả lỗi conflict rõ ràng, không âm thầm ghi đè.
- Mô tả cụ thể: biên tập viên A và B cùng mở bài ở version 5, B lưu trước (bài chuyển sang version 6), A lưu sau vẫn gửi version 5 — request của A phải bị từ chối với thông báo "bài viết đã được người khác cập nhật", kèm hiển thị nội dung mới nhất (version 6) để A quyết định merge tay hoặc ghi đè có ý thức, không tự động merge ngầm.
- Cho phép A chọn "lưu đè" sau khi đã xem nội dung mới của B (ghi đè có ý thức, không phải ghi đè mù), hành động này phải tạo version mới dựa trên version hiện tại thực (6 → 7), không dùng lại version cũ (5) đã lỗi thời.
- Tự động lưu draft định kỳ (auto-save) trong lúc biên tập viên đang soạn không được tính là 1 "lưu chính thức" làm tăng version theo cách gây xung đột giả với người khác — quy định rõ auto-save lưu vào 1 bản riêng (draft riêng theo user) tách khỏi bản chính thức, chỉ khi publish mới áp dụng optimistic lock lên bản chính thức.
- Ghi log lịch sử các version bài viết (ai sửa, khi nào, nội dung gì thay đổi) để biên tập viên có thể xem lại/khôi phục version cũ nếu bị lưu đè nhầm.

---

# Base ERD — CMS chưa có version, lưu bài viết ghi đè mù

Đây là **base**: trạng thái CMS *trước khi* áp dụng optimistic locking theo version. Suy luận từ đề bài, base đã có `EDITOR` (biên tập viên) và `ARTICLE` (bài viết) với nội dung cơ bản, nhiều editor có thể cùng mở 1 bài viết. Base chưa có cột `version`, chưa có lịch sử version, và mỗi lần lưu chỉ ghi đè trực tiếp `content` — ai lưu sau sẽ ghi đè hoàn toàn thay đổi của người lưu trước mà không phát hiện được.

```mermaid
erDiagram
    EDITOR ||--o{ ARTICLE : "cập nhật gần nhất"

    EDITOR {
        string id PK
        string name
    }
    ARTICLE {
        string id PK
        string title
        string content
        string updated_by FK
        datetime updated_at
    }
```

---

# Base sequence — Mở bài viết để sửa (không có khái niệm version)

Đây là **base**, flow mở bài viết hiện tại: editor mở bài, nhận toàn bộ nội dung hiện có, không có thông tin version đi kèm vì hệ thống chưa track version. Đây là tiền đề để so sánh với yêu cầu 1 (thêm cột version phải trả về khi mở bài để client gửi kèm lúc lưu).

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    actor EditorB as Editor B
    participant CMS as CMS Service
    participant DB as ARTICLE table

    EditorA->>CMS: Mở bài viết X để sửa
    CMS->>DB: SELECT content FROM ARTICLE WHERE id = X
    DB-->>CMS: content hiện tại
    CMS-->>EditorA: Trả nội dung bài viết (không có version)

    EditorB->>CMS: Mở cùng bài viết X để sửa
    CMS->>DB: SELECT content FROM ARTICLE WHERE id = X
    DB-->>CMS: content hiện tại (giống hệt A vừa đọc)
    CMS-->>EditorB: Trả nội dung bài viết (không có version)
    Note over EditorA,EditorB: Cả 2 editor đang cùng sửa trên 1 bản nội dung như nhau, không ai biết người kia cũng đang mở bài
```

---

# Base sequence — Lưu bài viết (ghi đè mù, mất dữ liệu của người lưu trước)

Đây là **base**, flow lưu bài viết ở trạng thái hiện tại: không kiểm tra version, ai lưu sau sẽ ghi đè toàn bộ `content` của người lưu trước, kể cả khi 2 người sửa 2 phần nội dung khác nhau. Đây chính là vấn đề "lost update" mà toàn bộ đề bài (optimistic lock theo version) phải giải quyết.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    actor EditorB as Editor B
    participant CMS as CMS Service
    participant DB as ARTICLE table

    EditorB->>CMS: Lưu bài viết X với nội dung đã sửa
    CMS->>DB: UPDATE ARTICLE SET content = ... WHERE id = X
    DB-->>CMS: OK
    CMS-->>EditorB: Lưu thành công

    EditorA->>CMS: Lưu bài viết X với nội dung đã sửa (dựa trên bản cũ trước khi B lưu)
    CMS->>DB: UPDATE ARTICLE SET content = ... WHERE id = X
    DB-->>CMS: OK (ghi đè lên bản B vừa lưu, không phát hiện xung đột)
    CMS-->>EditorA: Lưu thành công
    Note over DB,EditorA: Toàn bộ thay đổi của Editor B đã bị mất mà không ai được cảnh báo, kể cả khi A và B sửa 2 đoạn khác nhau
```

---

# Enhance ERD — Version, optimistic lock, draft riêng theo user, lịch sử version

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm mới:

- `ARTICLE.version` — tăng dần mỗi lần lưu chính thức, đáp ứng yêu cầu 1 và yêu cầu 2 (điều kiện `WHERE version = 5` khi lưu, phát hiện xung đột).
- `ARTICLE_VERSION_HISTORY` (mới) — snapshot nội dung mỗi version, ai sửa/khi nào, đáp ứng yêu cầu 5 (xem lại/khôi phục version cũ). Cũng là nguồn dữ liệu để hiển thị nội dung version mới nhất khi conflict xảy ra (yêu cầu 2, 3).
- `ARTICLE_DRAFT` (mới) — draft riêng theo từng editor, tách biệt hoàn toàn khỏi `ARTICLE.version` chính thức, đáp ứng yêu cầu 4 (auto-save không được tính là lưu chính thức, không gây xung đột giả).

```mermaid
erDiagram
    EDITOR ||--o{ ARTICLE : "cập nhật gần nhất"
    ARTICLE ||--o{ ARTICLE_VERSION_HISTORY : "có lịch sử version"
    ARTICLE ||--o{ ARTICLE_DRAFT : "có draft riêng theo user"
    EDITOR ||--o{ ARTICLE_DRAFT : "sở hữu draft của mình"

    EDITOR {
        string id PK
        string name
    }
    ARTICLE {
        string id PK
        string title
        string content
        int version "tăng dần mỗi lần lưu chính thức/publish"
        string updated_by FK
        datetime updated_at
    }
    ARTICLE_VERSION_HISTORY {
        string id PK
        string article_id FK
        int version
        string content_snapshot
        string edited_by FK
        datetime edited_at
        string change_summary
    }
    ARTICLE_DRAFT {
        string id PK
        string article_id FK
        string editor_id FK
        string draft_content
        datetime saved_at
    }
```

---

# Enhance sequence — Save article (optimistic lock theo version, ghi lịch sử)

Đây là **enhance** của flow `save-article` đã có ở base. So với base (ghi đè mù, không kiểm tra gì), giờ mọi request lưu phải gửi kèm `version` đã đọc, server chỉ chấp nhận nếu version hiện tại trong DB còn khớp (`UPDATE ... WHERE version = 5`), đồng thời ghi snapshot vào lịch sử. Đáp ứng trực tiếp yêu cầu 1 (trường hợp lưu thành công, không có xung đột) và tạo dữ liệu cho yêu cầu 5 (lịch sử version).

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    participant CMS as CMS Service
    participant DB as ARTICLE table
    participant History as ARTICLE_VERSION_HISTORY

    EditorA->>CMS: Mở bài viết X, nhận content + version = 5
    EditorA->>CMS: Sửa nội dung, bấm Lưu (gửi kèm version = 5 đã đọc)
    CMS->>DB: UPDATE ARTICLE SET content = ..., version = 6 WHERE id = X AND version = 5
    DB-->>CMS: 1 row affected (version hiện tại đúng là 5, update thành công)
    CMS->>History: Ghi ARTICLE_VERSION_HISTORY (version=6, content_snapshot, edited_by=A, edited_at)
    CMS-->>EditorA: Lưu thành công, version mới = 6
    Note over DB,History: Vì không ai khác lưu xen giữa, điều kiện WHERE version = 5 khớp nên update đi qua bình thường
```

---

# Enhance sequence — Phát hiện xung đột khi lưu (version đã lỗi thời)

Đây là **enhance**, flow mới mô tả đúng kịch bản nêu trong yêu cầu 2 của đề bài: Editor A và B cùng mở bài ở version 5, B lưu trước (bài chuyển version 6), A lưu sau vẫn gửi version 5 — request của A phải bị từ chối rõ ràng, kèm hiển thị nội dung mới nhất (version 6), không tự động ghi đè.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    actor EditorB as Editor B
    participant CMS as CMS Service
    participant DB as ARTICLE table

    EditorA->>CMS: Mở bài viết X (version = 5)
    EditorB->>CMS: Mở cùng bài viết X (version = 5)

    EditorB->>CMS: Lưu bài (gửi kèm version = 5)
    CMS->>DB: UPDATE ARTICLE SET content = ..., version = 6 WHERE id = X AND version = 5
    DB-->>CMS: 1 row affected, thành công
    CMS-->>EditorB: Lưu thành công, version mới = 6

    EditorA->>CMS: Lưu bài (vẫn gửi kèm version = 5 đã đọc từ đầu)
    CMS->>DB: UPDATE ARTICLE SET content = ..., version = 6 WHERE id = X AND version = 5
    DB-->>CMS: 0 row affected (version hiện tại trong DB đã là 6, không còn là 5)
    CMS->>DB: SELECT content, version FROM ARTICLE WHERE id = X
    DB-->>CMS: content mới nhất, version = 6
    CMS-->>EditorA: Từ chối lưu, báo "bài viết đã được người khác cập nhật", kèm nội dung version 6 mới nhất
    Note over EditorA,CMS: A phải tự quyết định merge tay hoặc ghi đè có ý thức, hệ thống không tự động merge ngầm
```

---

# Enhance sequence — Lưu đè có ý thức sau khi xem nội dung mới

Đây là **enhance**, flow tiếp nối trực tiếp sau `save-conflict-detected`, đáp ứng yêu cầu 3 của đề bài: sau khi Editor A đã xem nội dung mới nhất (version 6) của Editor B, A chọn "lưu đè có ý thức" — hành động này phải tạo version mới dựa trên version hiện tại thực (6 → 7), không dùng lại version cũ (5) đã lỗi thời.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    participant CMS as CMS Service
    participant DB as ARTICLE table
    participant History as ARTICLE_VERSION_HISTORY

    Note over EditorA: A vừa nhận thông báo conflict, đã xem nội dung version 6 của B
    EditorA->>CMS: Chọn "Lưu đè" (áp nội dung A đã soạn lên trên bản version 6 mới nhất)
    CMS->>DB: SELECT version FROM ARTICLE WHERE id = X
    DB-->>CMS: version hiện tại = 6
    CMS->>DB: UPDATE ARTICLE SET content = <nội dung A>, version = 7 WHERE id = X AND version = 6
    DB-->>CMS: 1 row affected, thành công (dựa trên version thực 6, không dùng version cũ 5)
    CMS->>History: Ghi ARTICLE_VERSION_HISTORY (version=7, content_snapshot, edited_by=A, change_summary="ghi đè có ý thức sau conflict với version 6")
    CMS-->>EditorA: Lưu đè thành công, version mới = 7
    Note over CMS,History: Vì lấy đúng version hiện tại thực (6) làm mốc, hành động lưu đè không bị optimistic lock chặn lại lần 2 và vẫn giữ được vết lịch sử ai đã ghi đè lên ai
```

---

# Enhance sequence — Auto-save draft riêng theo user, không tăng version chính thức

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 4 của đề bài: trong lúc Editor đang soạn, hệ thống tự động lưu draft định kỳ vào `ARTICLE_DRAFT` riêng theo từng user, hoàn toàn tách khỏi `ARTICLE.version` chính thức — không gây xung đột giả với người khác đang sửa cùng bài.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    participant Client as CMS Client (auto-save timer)
    participant CMS as CMS Service
    participant Draft as ARTICLE_DRAFT (riêng theo editor_id)
    participant DB as ARTICLE table (bản chính thức)

    EditorA->>Client: Đang soạn nội dung bài viết X
    loop Mỗi 30 giây (auto-save)
        Client->>CMS: Auto-save draft hiện tại (article_id=X, editor_id=A)
        CMS->>Draft: UPSERT ARTICLE_DRAFT (article_id=X, editor_id=A, draft_content, saved_at)
        Draft-->>CMS: OK
        Note over DB: ARTICLE.version của bản chính thức hoàn toàn không đổi
    end
    EditorA->>Client: Bấm "Publish" (lưu chính thức)
    Client->>CMS: Publish với version đã đọc ban đầu
    CMS->>DB: UPDATE ARTICLE ... WHERE version = <version đã đọc> (áp dụng optimistic lock như flow save-article)
    DB-->>CMS: Kết quả tuỳ còn khớp version hay không
    CMS-->>EditorA: Chỉ lúc publish mới có khả năng bị conflict, auto-save trước đó không tạo xung đột nào
```

---

# Enhance sequence — Xem lịch sử version và khôi phục bản cũ

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 5 của đề bài: biên tập viên có thể xem lại lịch sử các version bài viết (ai sửa, khi nào, thay đổi gì) và khôi phục 1 version cũ nếu phát hiện bị lưu đè nhầm.

```mermaid
sequenceDiagram
    actor Editor
    participant CMS as CMS Service
    participant History as ARTICLE_VERSION_HISTORY
    participant DB as ARTICLE table

    Editor->>CMS: Xem lịch sử version của bài viết X
    CMS->>History: SELECT * FROM ARTICLE_VERSION_HISTORY WHERE article_id = X ORDER BY version DESC
    History-->>CMS: Danh sách version kèm ai sửa, khi nào, nội dung thay đổi
    CMS-->>Editor: Hiển thị timeline lịch sử version

    Editor->>CMS: Chọn khôi phục version 5 (nghi ngờ bị lưu đè nhầm ở version 7)
    CMS->>History: Lấy content_snapshot của version 5
    History-->>CMS: content_snapshot version 5
    CMS->>DB: SELECT version FROM ARTICLE WHERE id = X
    DB-->>CMS: version hiện tại = 7
    CMS->>DB: UPDATE ARTICLE SET content = <snapshot version 5>, version = 8 WHERE id = X AND version = 7
    DB-->>CMS: 1 row affected
    CMS->>History: Ghi ARTICLE_VERSION_HISTORY (version=8, change_summary="khôi phục nội dung từ version 5")
    CMS-->>Editor: Khôi phục thành công, bài viết giờ ở version 8 với nội dung của version 5
    Note over CMS,History: Việc khôi phục cũng đi qua optimistic lock như 1 lần lưu bình thường, và bản thân nó cũng được ghi vào lịch sử
```
