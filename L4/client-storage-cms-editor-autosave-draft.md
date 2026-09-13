# Autosave draft chống mất nội dung bằng IndexedDB cho trình soạn thảo CMS

**Hệ thống:** Trình soạn thảo nội dung dạng rich-text (CMS viết bài, hoặc công cụ soạn tài liệu) cho phép soạn bài dài, cần autosave draft liên tục vào IndexedDB để không mất nội dung khi crash trình duyệt/mất điện, trước khi kịp lưu lên server.

**Vai trò của flow:** Đảm bảo "không bao giờ mất nội dung đang gõ" mà không làm chậm trải nghiệm gõ phím, và không tạo xung đột khi cùng lúc có bản draft cục bộ và bản đã lưu trên server.

**Yêu cầu cụ thể:**
- Autosave vào IndexedDB theo debounce (ví dụ mỗi 2-3 giây sau khi ngừng gõ) thay vì lưu mỗi keystroke, để không chặn main thread khi nội dung dài (hàng chục nghìn từ) — đo rõ ngưỡng độ dài nội dung mà việc serialize/ghi IndexedDB bắt đầu gây giật khi gõ.
- Khi mở lại app sau khi crash/đóng nhầm tab, phải phát hiện có draft chưa lưu trong IndexedDB mới hơn bản đã lưu trên server, và hỏi rõ user chọn khôi phục draft cục bộ hay giữ bản trên server — không tự động ghi đè một trong hai một cách ngầm định.
- Mở cùng 1 bài viết ở 2 tab/2 máy khác nhau: draft cục bộ trong IndexedDB chỉ có ý nghĩa cục bộ (không đồng bộ qua lại giữa các máy), nên khi server đã có bản mới hơn (do sửa từ máy/tab khác) phải cảnh báo rõ nguy cơ mất nội dung nếu cứ lưu đè, thay vì im lặng ghi đè draft cũ hơn.
- Dọn dẹp draft trong IndexedDB sau khi đã lưu thành công lên server hoặc bài viết đã publish/xóa, tránh tồn đọng draft rác vô thời hạn làm phình dung lượng IndexedDB và có thể vô tình khôi phục nhầm draft đã lỗi thời cho một bài viết đã bị xóa từ lâu.
- Xử lý khi IndexedDB bị lỗi/không khả dụng (một số trình duyệt chặn trong chế độ ẩn danh hoặc storage bị corrupt) — có fallback về autosave lên server với tần suất thấp hơn, và cảnh báo rõ cho user biết chế độ "an toàn khi crash cục bộ" đang không hoạt động.

---

# Base ERD — Trình soạn thảo CMS trước khi có autosave draft bằng IndexedDB

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống **trước khi** có cơ chế autosave draft cục bộ. Đề bài nói tới trình soạn thảo rich-text lưu bài lên server, nên base chỉ cần User và Article ở phía server, lưu trực tiếp khi bấm Save hoặc autosave định kỳ thẳng lên server. Chưa có bất kỳ entity nào phía client (IndexedDB draft, đo hiệu năng gõ, trạng thái storage) — toàn bộ phần đó là enhance.

```mermaid
erDiagram
    USER ||--o{ ARTICLE : authors

    USER {
        string id PK
        string display_name
    }
    ARTICLE {
        string id PK
        string author_id FK
        string title
        string content
        string status "draft | published | deleted"
        datetime updated_at
    }
```

---

# Base sequence — Edit article (lưu thẳng lên server, chưa có autosave cục bộ)

Đây là **base**, flow soạn thảo bài viết ở trạng thái trước enhance. Người dùng gõ nội dung, việc lưu chỉ xảy ra khi bấm nút Save thủ công hoặc autosave định kỳ gọi thẳng API server, không có bước lưu tạm cục bộ nào. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi autosave chuyển sang debounce ghi vào IndexedDB trước.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant API as Article API
    participant DB as Server DB

    User->>Editor: Gõ nội dung liên tục
    loop Autosave định kỳ mỗi X giây, không phân biệt độ dài nội dung
        Editor->>API: POST toàn bộ content hiện tại
        API->>DB: Cập nhật ARTICLE.content, updated_at
        API-->>Editor: Xác nhận đã lưu
    end
    User->>Editor: Bấm nút Save thủ công
    Editor->>API: POST content hiện tại
    API->>DB: Cập nhật ARTICLE.content, updated_at
    API-->>Editor: Xác nhận đã lưu
```

---

# Base sequence — Open article (chưa có kiểm tra draft cục bộ)

Đây là **base**, flow mở bài viết để soạn tiếp. Trình soạn thảo chỉ đơn giản tải bản mới nhất từ server và hiển thị, không có khái niệm draft cục bộ nên không có bước so sánh hay hỏi người dùng chọn phiên bản nào. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi việc mở bài viết phải kiểm tra thêm draft trong IndexedDB.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant API as Article API
    participant DB as Server DB

    User->>Editor: Mở bài viết để soạn tiếp
    Editor->>API: GET article theo id
    API->>DB: Lấy bản ghi ARTICLE mới nhất
    DB-->>API: Trả về content, updated_at
    API-->>Editor: Trả về nội dung bài viết
    Editor-->>User: Hiển thị nội dung để soạn tiếp
```

---

# Enhance ERD — Autosave draft cục bộ bằng IndexedDB

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (chỉ có USER/ARTICLE ở server), phần enhance thêm 3 entity mới hoàn toàn ở phía client/observability, ứng trực tiếp với các yêu cầu trong đề bài:

- `LOCAL_DRAFT` (mới, lưu trong IndexedDB) — snapshot content debounce mỗi 2-3 giây, kèm `content_length` và `updated_at_local` để so sánh với `ARTICLE.updated_at` trên server — phục vụ yêu cầu 1, 2, 3.
- `TYPING_PERF_METRIC` (mới) — đo thời gian serialize + ghi IndexedDB theo từng khoảng độ dài nội dung, để xác định ngưỡng bắt đầu gây giật — phục vụ yêu cầu 1.
- `STORAGE_HEALTH` (mới) — trạng thái IndexedDB khả dụng hay không theo từng phiên trình duyệt, và có đang ở fallback mode hay không — phục vụ yêu cầu 5.

`LOCAL_DRAFT` có vòng đời gắn với 1 `ARTICLE`, bị xoá khi lưu server thành công hoặc bài viết publish/xoá (yêu cầu 4).

```mermaid
erDiagram
    USER ||--o{ ARTICLE : authors
    ARTICLE ||--o| LOCAL_DRAFT : "may have local draft (IndexedDB)"
    ARTICLE ||--o{ TYPING_PERF_METRIC : "measured while editing"
    USER ||--o{ STORAGE_HEALTH : "has per browser session"

    USER {
        string id PK
        string display_name
    }
    ARTICLE {
        string id PK
        string author_id FK
        string title
        string content
        string status "draft | published | deleted"
        datetime updated_at
    }
    LOCAL_DRAFT {
        string id PK
        string article_id FK
        string content_snapshot
        int content_length
        string device_id
        string save_trigger "debounce | manual"
        datetime updated_at_local
    }
    TYPING_PERF_METRIC {
        string id PK
        string article_id FK
        int content_length_bucket
        int serialize_ms
        int write_ms
        boolean jank_detected
        datetime measured_at
    }
    STORAGE_HEALTH {
        string id PK
        string user_id FK
        string session_id
        boolean indexeddb_available
        boolean fallback_mode_active
        datetime detected_at
    }
```

---

# Enhance sequence — Edit article (debounce autosave vào IndexedDB, đo ngưỡng gây giật)

Đây là **enhance**, cùng flow "edit-article" như ở base nhưng thay đổi hoàn toàn cơ chế autosave: thay vì gọi thẳng server định kỳ, mỗi lần ngừng gõ 2-3 giây mới serialize và ghi 1 bản draft vào IndexedDB (debounce), đồng thời đo thời gian serialize/ghi theo độ dài nội dung để xác định ngưỡng bắt đầu gây giật khi gõ. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant Debounce as Debounce Timer
    participant IDB as IndexedDB (LOCAL_DRAFT)
    participant Metric as TYPING_PERF_METRIC

    User->>Editor: Gõ liên tục (keystroke)
    loop Mỗi keystroke
        Editor->>Debounce: Reset timer debounce 2-3 giây
    end
    Note over Debounce: Chỉ khi người dùng NGỪNG gõ đủ 2-3 giây, timer mới bắn
    Debounce->>Editor: Timer hết hạn, kích hoạt autosave cục bộ
    Editor->>Editor: Serialize nội dung hiện tại, đo serialize_ms
    Editor->>IDB: Ghi LOCAL_DRAFT mới (content_snapshot, content_length, updated_at_local), đo write_ms
    IDB-->>Editor: Ghi thành công
    Editor->>Metric: Ghi nhận content_length_bucket, serialize_ms, write_ms

    alt Tổng serialize_ms + write_ms vượt ngưỡng gây giật (ví dụ nội dung hàng chục nghìn từ)
        Metric->>Metric: Đánh dấu jank_detected=true cho bucket độ dài này
        Note over Metric: Dữ liệu này dùng để xác định ngưỡng nên tối ưu, ví dụ serialize từng phần thay vì toàn bộ document
    else Trong ngưỡng bình thường
        Metric->>Metric: jank_detected=false
    end
```

---

# Enhance sequence — Open article (phát hiện draft cục bộ mới hơn bản server)

Đây là **enhance**, cùng flow "open-article" như ở base nhưng thêm bước kiểm tra draft trong IndexedDB trước khi hiển thị. Nếu phát hiện có draft cục bộ mới hơn bản đã lưu trên server (dấu hiệu của crash hoặc đóng nhầm tab trước đó), hệ thống phải hỏi rõ người dùng chọn khôi phục draft cục bộ hay giữ bản trên server, không tự động ghi đè ngầm định. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant IDB as IndexedDB (LOCAL_DRAFT)
    participant API as Article API

    User->>Editor: Mở lại bài viết (sau crash hoặc đóng nhầm tab)
    Editor->>API: GET article theo id
    API-->>Editor: Trả về content, updated_at (bản server)
    Editor->>IDB: Kiểm tra có LOCAL_DRAFT nào cho article này không

    alt Không có draft cục bộ, hoặc draft cục bộ cũ hơn/bằng bản server
        Editor-->>User: Hiển thị thẳng bản server, không cần hỏi gì thêm
    else Có draft cục bộ với updated_at_local mới hơn ARTICLE.updated_at trên server
        Editor-->>User: Hiển thị hộp thoại, phát hiện có nội dung chưa lưu mới hơn, khôi phục draft cục bộ hay giữ bản trên server
        alt User chọn khôi phục draft cục bộ
            Editor->>IDB: Đọc content_snapshot từ LOCAL_DRAFT
            Editor-->>User: Hiển thị nội dung từ draft cục bộ để soạn tiếp
        else User chọn giữ bản trên server
            Editor-->>User: Hiển thị nội dung từ server, bỏ qua draft cục bộ
            Note over Editor,IDB: Draft cục bộ cũ vẫn còn tồn tại tạm thời, sẽ được dọn ở flow cleanup
        end
    end
```

---

# Enhance sequence — Cảnh báo xung đột khi server đã có bản mới hơn từ tab/máy khác

Đây là **enhance**, flow hoàn toàn mới, xử lý tình huống mở cùng 1 bài viết ở 2 tab hoặc 2 máy khác nhau. Vì draft cục bộ trong IndexedDB chỉ có ý nghĩa cục bộ, không đồng bộ qua lại giữa các máy, nên trước khi lưu đè lên server hệ thống phải kiểm tra xem server đã bị sửa từ nơi khác mới hơn baseline mà tab hiện tại đang biết hay chưa, và cảnh báo rõ nguy cơ mất nội dung thay vì im lặng ghi đè. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài (Tab A)
    participant EditorA as Editor Tab A
    participant IDB as IndexedDB (LOCAL_DRAFT, cục bộ theo máy)
    participant API as Article API
    actor OtherUser as Người soạn bài (Tab B / máy khác)

    Note over EditorA: Tab A đang giữ baseline updated_at của bài viết từ lúc mở
    OtherUser->>API: Lưu thay đổi từ Tab B / máy khác, ARTICLE.updated_at được cập nhật mới hơn

    User->>EditorA: Tiếp tục gõ ở Tab A, tới lúc autosave/lưu
    EditorA->>IDB: Ghi LOCAL_DRAFT mới (chỉ cục bộ trên máy này)
    EditorA->>API: Trước khi lưu lên server, kiểm tra ARTICLE.updated_at hiện tại
    API-->>EditorA: Trả về updated_at mới hơn baseline mà Tab A đang có

    alt updated_at server mới hơn baseline của Tab A
        EditorA-->>User: Cảnh báo rõ, bài viết đã được sửa ở nơi khác, lưu đè có thể làm mất nội dung đó
        alt User chọn vẫn lưu đè
            EditorA->>API: Ghi đè content từ Tab A lên server
        else User chọn xem lại/hủy lưu
            EditorA-->>User: Không lưu, để user tự quyết định merge thủ công
        end
    else Không có xung đột, server vẫn là baseline cũ
        EditorA->>API: Lưu bình thường, không cần cảnh báo
    end
```

---

# Enhance sequence — Dọn dẹp draft cục bộ sau khi lưu server hoặc publish/xoá bài

Đây là **enhance**, flow hoàn toàn mới, đảm bảo draft trong IndexedDB không tồn đọng vô thời hạn. Ngay sau khi lưu thành công lên server, hoặc khi bài viết được publish/xoá, draft cục bộ tương ứng phải được xoá để tránh phình dung lượng IndexedDB và tránh vô tình khôi phục nhầm draft lỗi thời cho một bài đã bị xoá từ lâu. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant API as Article API
    participant IDB as IndexedDB (LOCAL_DRAFT)

    alt Lưu thành công lên server (thủ công hoặc sau khi resolve xung đột)
        Editor->>API: Lưu content lên server
        API-->>Editor: Xác nhận lưu thành công, trả về updated_at mới
        Editor->>IDB: Xoá LOCAL_DRAFT của article này vì đã đồng bộ lên server
    else User bấm Publish
        Editor->>API: Publish bài viết
        API-->>Editor: Xác nhận ARTICLE.status=published
        Editor->>IDB: Xoá LOCAL_DRAFT của article này
    else User hoặc hệ thống xoá bài viết
        Editor->>API: Xoá bài viết
        API-->>Editor: Xác nhận ARTICLE.status=deleted
        Editor->>IDB: Xoá LOCAL_DRAFT của article này, tránh khôi phục nhầm cho bài đã xoá
    end

    Note over IDB: Định kỳ cũng quét các LOCAL_DRAFT không còn ARTICLE tương ứng hoặc quá cũ, để dọn rác còn sót
```

---

# Enhance sequence — Fallback khi IndexedDB lỗi/không khả dụng

Đây là **enhance**, flow hoàn toàn mới, xử lý trường hợp IndexedDB bị chặn (ví dụ chế độ ẩn danh của một số trình duyệt) hoặc storage bị corrupt. Hệ thống phải phát hiện tình trạng này, chuyển sang fallback autosave thẳng lên server với tần suất thấp hơn, và cảnh báo rõ cho người dùng biết chế độ "an toàn khi crash cục bộ" đang không hoạt động. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant IDB as IndexedDB
    participant Health as STORAGE_HEALTH
    participant API as Article API

    Editor->>IDB: Thử mở/ghi thử một bản ghi kiểm tra khi khởi tạo editor
    alt IndexedDB khả dụng bình thường
        IDB-->>Editor: Ghi/đọc thành công
        Editor->>Health: Ghi indexeddb_available=true, fallback_mode_active=false
        Note over Editor: Tiếp tục dùng debounce autosave vào IndexedDB như flow edit-article
    else IndexedDB bị chặn hoặc lỗi/corrupt
        IDB-->>Editor: Lỗi khi mở database hoặc ghi thử thất bại
        Editor->>Health: Ghi indexeddb_available=false, fallback_mode_active=true
        Editor-->>User: Hiển thị cảnh báo rõ ràng, chế độ an toàn khi crash cục bộ đang không hoạt động
        loop Autosave fallback với tần suất thấp hơn nhiều so với debounce bình thường
            Editor->>API: Gửi thẳng content hiện tại lên server, chấp nhận tần suất thưa hơn
            API-->>Editor: Xác nhận đã lưu tạm trên server
        end
    end
```
