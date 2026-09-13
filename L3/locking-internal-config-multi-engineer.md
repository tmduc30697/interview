# Chỉnh sửa cấu hình hệ thống nội bộ (feature flag, config) bởi nhiều kỹ sư

**Hệ thống:** Một dashboard nội bộ cho kỹ sư quản lý feature flag/config runtime của hệ thống production, thay đổi có hiệu lực ngay khi lưu, nhiều kỹ sư ở các team khác nhau có quyền chỉnh sửa.

**Vai trò của flow:** Vì thay đổi config có tác động ngay tới production và tần suất nhiều kỹ sư cùng sửa 1 config trong cùng thời điểm là thấp nhưng hậu quả nếu ghi đè nhầm khá nghiêm trọng, flow nên dùng optimistic locking kết hợp với hiển thị rõ diff thay đổi trước khi cho phép lưu đè.

**Yêu cầu cụ thể:**
- Mỗi config có version, khi kỹ sư A lưu thay đổi dựa trên version cũ mà đã có kỹ sư B lưu thay đổi khác trước đó (version mới hơn), hệ thống phải từ chối lưu và hiển thị rõ diff giữa bản A đang muốn lưu, bản gốc A đã đọc, và bản mới nhất B đã lưu — để A tự quyết định merge tay chính xác, không tự động merge ngầm với config nhạy cảm ảnh hưởng production.
- Mô tả cụ thể: kỹ sư A đang tắt feature flag X (do phát hiện lỗi khẩn cấp), đúng lúc kỹ sư B đang sửa 1 config khác không liên quan trong cùng bộ config đó nhưng version chung bị tăng do B lưu trước — cần đánh giá xem có nên tách optimistic lock ở cấp độ từng flag/config riêng lẻ (để A không bị chặn bởi thay đổi không liên quan của B) thay vì version chung cho cả nhóm config, đặc biệt quan trọng cho action khẩn cấp như tắt flag khi có sự cố.
- Với action khẩn cấp (ví dụ "tắt flag ngay" trong sự cố production), nên có đường xử lý riêng ưu tiên ghi đè ngay không chờ optimistic lock thông thường (bỏ qua version check hoặc có nút "buộc lưu" rõ ràng có cảnh báo), vì trong tình huống khẩn cấp việc bị chặn bởi conflict version có thể gây chậm trễ nguy hiểm hơn rủi ro ghi đè nhầm — nhưng phải log rõ ràng và cảnh báo action này đã bỏ qua kiểm tra xung đột.
- Mọi thay đổi config (dù qua optimistic lock thông thường hay đường khẩn cấp) phải được ghi lịch sử đầy đủ và không thể xóa (audit log immutable): ai, khi nào, giá trị trước/sau, có phải action khẩn cấp không — để phục vụ điều tra sau sự cố nếu 1 thay đổi config gây ảnh hưởng không mong muốn.
- Có cơ chế thông báo real-time cho các kỹ sư khác đang mở màn hình chỉnh sửa cùng 1 config khi có ai đó vừa lưu thay đổi (qua WebSocket/polling), để giảm khả năng họ submit dựa trên version đã lỗi thời và phải xử lý conflict muộn hơn cần thiết.

---

# Base ERD — Config dashboard dùng optimistic lock ở cấp cả nhóm config

Đây là **base**: trạng thái dashboard config nội bộ *trước khi* tách optimistic lock theo từng flag/config riêng lẻ. Suy luận từ đề bài (yêu cầu 2 mô tả rõ "version chung bị tăng do B lưu trước" như cách làm hiện tại), base đã có `ENGINEER`, `CONFIG_GROUP` (1 bộ config runtime liên quan nhau, vd cùng 1 service), và các `CONFIG_ITEM` (từng flag/config riêng lẻ) nằm trong group đó — nhưng cột `version` để optimistic lock nằm ở cấp `CONFIG_GROUP` chung cho cả nhóm, chưa có version riêng từng item. Base cũng chưa có đường xử lý khẩn cấp, chưa có audit log immutable, và chưa có cơ chế thông báo real-time.

```mermaid
erDiagram
    ENGINEER ||--o{ CONFIG_GROUP : "cập nhật gần nhất"
    CONFIG_GROUP ||--o{ CONFIG_ITEM : "chứa"

    ENGINEER {
        string id PK
        string name
        string team
    }
    CONFIG_GROUP {
        string id PK
        string name "vd checkout-service-config"
        int version "version chung cho cả nhóm config"
        string updated_by FK
        datetime updated_at
    }
    CONFIG_ITEM {
        string id PK
        string group_id FK
        string key "vd feature_flag_x"
        string value
        string type "feature_flag|config"
    }
```

---

# Base sequence — Update config (optimistic lock ở cấp cả nhóm, thành công)

Đây là **base**, flow lưu thay đổi config hiện tại: optimistic lock áp dụng cho toàn bộ `CONFIG_GROUP`, chỉ 1 cột `version` chung cho mọi config/flag trong nhóm. Khi chỉ có 1 kỹ sư sửa tại 1 thời điểm, flow hoạt động bình thường — đây là tiền đề để so sánh với flow tiếp theo khi 2 kỹ sư sửa 2 config không liên quan trong cùng nhóm.

```mermaid
sequenceDiagram
    actor EngineerB as Kỹ sư B
    participant Dash as Config Dashboard
    participant DB as CONFIG_GROUP table

    EngineerB->>Dash: Mở nhóm config, nhận version = 12
    EngineerB->>Dash: Sửa 1 config không liên quan tới flag khẩn cấp, gửi kèm version = 12
    Dash->>DB: UPDATE CONFIG_GROUP SET ..., version = 13 WHERE id = G AND version = 12
    DB-->>Dash: 1 row affected, thành công
    Dash-->>EngineerB: Lưu thành công, version mới = 13
    Note over Dash,DB: Vì chỉ 1 người sửa tại 1 thời điểm, version chung chưa gây vấn đề gì
```

---

# Base sequence — Xung đột giả chặn cả thao tác khẩn cấp

Đây là **base**, mô tả đúng kịch bản nêu ở yêu cầu 2 của đề bài: kỹ sư A đang tắt feature flag X do phát hiện lỗi khẩn cấp, đúng lúc kỹ sư B đang sửa 1 config khác không liên quan trong cùng nhóm nhưng version chung bị tăng do B lưu trước. Vì optimistic lock ở cấp cả nhóm và base chưa có đường xử lý khẩn cấp, A bị chặn hoàn toàn — đây chính là vấn đề nghiêm trọng mà việc tách version theo từng item và thêm đường khẩn cấp (enhance) phải giải quyết.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A (đang xử lý sự cố)
    actor EngineerB as Kỹ sư B (sửa config không liên quan)
    participant Dash as Config Dashboard
    participant DB as CONFIG_GROUP table

    EngineerA->>Dash: Mở nhóm config, nhận version = 12 (đang định tắt flag X)
    EngineerB->>Dash: Mở cùng nhóm config, cũng nhận version = 12

    EngineerB->>Dash: Lưu thay đổi config khác (không liên quan flag X), gửi version = 12
    Dash->>DB: UPDATE CONFIG_GROUP SET ..., version = 13 WHERE id = G AND version = 12
    DB-->>Dash: 1 row affected, thành công
    Dash-->>EngineerB: Lưu thành công, version mới = 13

    EngineerA->>Dash: Tắt flag X khẩn cấp, gửi version = 12 (đã lỗi thời do B vừa lưu)
    Dash->>DB: UPDATE CONFIG_GROUP SET flag_x = off, version = 13 WHERE id = G AND version = 12
    DB-->>Dash: 0 row affected (version hiện tại đã là 13)
    Dash-->>EngineerA: Từ chối lưu, báo "config đã được người khác cập nhật"
    Note over EngineerA,Dash: A bị chặn tắt flag khẩn cấp chỉ vì B lưu 1 config hoàn toàn không liên quan, và base chưa có đường xử lý khẩn cấp nào để bỏ qua tình huống này
```

---

# Enhance ERD — Version theo từng config item, đường khẩn cấp, audit log immutable, notify real-time

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (`version` chung ở `CONFIG_GROUP`), thay đổi và bổ sung:

- `version` chuyển xuống cấp `CONFIG_ITEM` (mỗi flag/config riêng lẻ có version riêng) — đáp ứng yêu cầu 1 (conflict theo đúng item) và yêu cầu 2 (A tắt flag X không còn bị chặn bởi thay đổi không liên quan của B).
- `CONFIG_ITEM` thêm `is_emergency_override`/liên kết tới log để phân biệt lần lưu nào đã bấm "buộc lưu" — đáp ứng yêu cầu 3.
- Entity mới `CONFIG_AUDIT_LOG` (immutable, append-only, không có API xóa/sửa) — ghi mọi thay đổi kể cả qua đường khẩn cấp, đáp ứng yêu cầu 4.
- Entity mới `CONFIG_EDIT_SESSION` — theo dõi kỹ sư nào đang mở màn hình sửa config nào, làm nền cho việc đẩy thông báo real-time qua WebSocket/polling, đáp ứng yêu cầu 5.

```mermaid
erDiagram
    ENGINEER ||--o{ CONFIG_ITEM : "cập nhật gần nhất"
    CONFIG_GROUP ||--o{ CONFIG_ITEM : "chứa"
    CONFIG_ITEM ||--o{ CONFIG_AUDIT_LOG : "có lịch sử thay đổi"
    CONFIG_ITEM ||--o{ CONFIG_EDIT_SESSION : "đang được xem/sửa bởi"
    ENGINEER ||--o{ CONFIG_EDIT_SESSION : "mở phiên xem/sửa"
    ENGINEER ||--o{ CONFIG_AUDIT_LOG : "thực hiện thay đổi"

    ENGINEER {
        string id PK
        string name
        string team
    }
    CONFIG_GROUP {
        string id PK
        string name "vd checkout-service-config"
    }
    CONFIG_ITEM {
        string id PK
        string group_id FK
        string key "vd feature_flag_x"
        string value
        string type "feature_flag|config"
        int version "version riêng cho từng item"
        string updated_by FK
        datetime updated_at
    }
    CONFIG_AUDIT_LOG {
        string id PK
        string config_item_id FK
        string changed_by FK
        datetime changed_at
        string old_value
        string new_value
        bool is_emergency_override "true nếu dùng đường buộc lưu bỏ qua version check"
        string note
    }
    CONFIG_EDIT_SESSION {
        string id PK
        string config_item_id FK
        string engineer_id FK
        int last_seen_version "version engineer này đang nhìn thấy trên UI"
        datetime connected_at
    }
```

---

# Enhance sequence — Update config (conflict theo đúng item, hiển thị diff 3 bên)

Đây là **enhance** của flow `update-config` đã có ở base. So với base (chỉ báo chung chung "đã được cập nhật"), khi version của đúng `CONFIG_ITEM` đang sửa bị lệch, hệ thống phải hiển thị rõ diff giữa bản kỹ sư A đang muốn lưu, bản gốc A đã đọc, và bản mới nhất kỹ sư B đã lưu — đáp ứng đúng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A
    actor EngineerB as Kỹ sư B
    participant Dash as Config Dashboard
    participant DB as CONFIG_ITEM table
    participant Log as CONFIG_AUDIT_LOG

    EngineerA->>Dash: Mở config "max_retry_count", nhận value=3, version=5
    EngineerB->>Dash: Mở cùng config "max_retry_count", cũng nhận value=3, version=5

    EngineerB->>Dash: Lưu value=5, gửi version=5
    Dash->>DB: UPDATE CONFIG_ITEM SET value=5, version=6 WHERE id=X AND version=5
    DB-->>Dash: 1 row affected, thành công
    Dash->>Log: Ghi CONFIG_AUDIT_LOG (old_value=3, new_value=5, changed_by=B, is_emergency_override=false)
    Dash-->>EngineerB: Lưu thành công, version mới = 6

    EngineerA->>Dash: Lưu value=10, vẫn gửi version=5 (đã lỗi thời)
    Dash->>DB: UPDATE CONFIG_ITEM SET value=10, version=6 WHERE id=X AND version=5
    DB-->>Dash: 0 row affected (version hiện tại đã là 6)
    Dash->>DB: SELECT value, version FROM CONFIG_ITEM WHERE id=X
    DB-->>Dash: value=5, version=6
    Dash-->>EngineerA: Từ chối lưu, hiển thị diff 3 bên, bản A muốn lưu (10), bản gốc A đã đọc (3), bản mới nhất B đã lưu (5)
    Note over EngineerA,Dash: A tự quyết định merge tay chính xác dựa trên 3 giá trị này, hệ thống không tự động merge ngầm với config nhạy cảm production
```

---

# Enhance sequence — Xung đột giả không còn xảy ra nhờ version theo từng item

Đây là **enhance** của flow `cross-flag-false-conflict-emergency` đã có ở base. So với base (A bị chặn tắt flag X chỉ vì B lưu 1 config không liên quan), giờ mỗi `CONFIG_ITEM` có version riêng nên B lưu config khác không hề ảnh hưởng tới version của flag X — A tắt được flag ngay qua optimistic lock bình thường, không cần tới đường khẩn cấp. Đáp ứng trực tiếp yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A (đang xử lý sự cố)
    actor EngineerB as Kỹ sư B (sửa config không liên quan)
    participant Dash as Config Dashboard
    participant FlagDB as CONFIG_ITEM "feature_flag_x"
    participant OtherDB as CONFIG_ITEM "other_config"

    EngineerA->>Dash: Mở feature_flag_x, nhận value=on, version=7
    EngineerB->>Dash: Mở other_config (không liên quan), nhận value=A, version=20

    EngineerB->>Dash: Lưu other_config, gửi version=20
    Dash->>OtherDB: UPDATE CONFIG_ITEM SET value=..., version=21 WHERE id=other_config AND version=20
    OtherDB-->>Dash: 1 row affected, thành công
    Dash-->>EngineerB: Lưu thành công

    EngineerA->>Dash: Tắt feature_flag_x khẩn cấp, gửi version=7
    Dash->>FlagDB: UPDATE CONFIG_ITEM SET value=off, version=8 WHERE id=feature_flag_x AND version=7
    FlagDB-->>Dash: 1 row affected, thành công (version của feature_flag_x hoàn toàn độc lập với other_config)
    Dash-->>EngineerA: Tắt flag thành công ngay lập tức qua optimistic lock bình thường
    Note over FlagDB,OtherDB: Vì mỗi config item có version riêng, thay đổi không liên quan của B không còn chặn được thao tác khẩn cấp của A
```

---

# Enhance sequence — Đường khẩn cấp "buộc lưu", bỏ qua version check

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 3 của đề bài: khi vẫn thực sự xảy ra conflict đúng ngay trên item cần tắt khẩn cấp (không chỉ là xung đột giả), kỹ sư có thể bấm "buộc lưu" để bỏ qua version check, ưu tiên tốc độ xử lý sự cố hơn rủi ro ghi đè nhầm — nhưng action này bắt buộc phải được log rõ ràng và cảnh báo.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A (đang xử lý sự cố nghiêm trọng)
    actor EngineerC as Kỹ sư C
    participant Dash as Config Dashboard
    participant FlagDB as CONFIG_ITEM "feature_flag_x"
    participant Log as CONFIG_AUDIT_LOG

    EngineerA->>Dash: Mở feature_flag_x, nhận value=on, version=9
    EngineerC->>Dash: Cũng vừa sửa feature_flag_x (vd đổi rollout %), lưu thành công, version=10

    EngineerA->>Dash: Tắt flag khẩn cấp, gửi version=9 (đã lỗi thời)
    Dash->>FlagDB: UPDATE CONFIG_ITEM SET value=off, version=10 WHERE id=feature_flag_x AND version=9
    FlagDB-->>Dash: 0 row affected (version hiện tại đã là 10, đây là conflict thật)
    Dash-->>EngineerA: Từ chối, hiển thị cảnh báo "config đã đổi, bạn có muốn BUỘC LƯU không (bỏ qua kiểm tra xung đột)?"

    EngineerA->>Dash: Xác nhận "Buộc lưu" (chấp nhận rủi ro vì đang là sự cố khẩn cấp)
    Dash->>FlagDB: UPDATE CONFIG_ITEM SET value=off, version=11 WHERE id=feature_flag_x (không kèm điều kiện version)
    FlagDB-->>Dash: 1 row affected, ghi đè thành công bất kể version hiện tại
    Dash->>Log: Ghi CONFIG_AUDIT_LOG (changed_by=A, old_value=<giá trị bị ghi đè>, new_value=off, is_emergency_override=true, note="bỏ qua version check do sự cố khẩn cấp")
    Dash-->>EngineerA: Flag đã tắt, kèm cảnh báo hiển thị lại rằng action này đã bỏ qua kiểm tra xung đột
    Note over Dash,Log: Vì action được đánh dấu is_emergency_override=true trong audit log, đội điều tra sau này luôn phân biệt được đâu là lưu bình thường, đâu là buộc lưu khẩn cấp
```

---

# Enhance sequence — Thông báo real-time khi có người vừa lưu thay đổi

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 5 của đề bài: các kỹ sư khác đang mở màn hình chỉnh sửa cùng 1 config được thông báo real-time (qua WebSocket) ngay khi có ai đó vừa lưu thay đổi, giảm khả năng họ submit dựa trên version đã lỗi thời.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A
    actor EngineerC as Kỹ sư C (đang mở cùng màn hình sửa)
    participant Dash as Config Dashboard
    participant WS as WebSocket Gateway
    participant SessionStore as CONFIG_EDIT_SESSION
    participant FlagDB as CONFIG_ITEM

    EngineerC->>Dash: Mở màn hình sửa feature_flag_x, nhận version=7
    Dash->>SessionStore: Ghi CONFIG_EDIT_SESSION (engineer=C, config_item=feature_flag_x, last_seen_version=7)
    Dash->>WS: Subscribe kênh cập nhật của feature_flag_x cho Engineer C

    EngineerA->>Dash: Lưu thành công feature_flag_x, version mới = 8
    Dash->>SessionStore: Tra các CONFIG_EDIT_SESSION đang mở feature_flag_x có last_seen_version cũ hơn 8
    SessionStore-->>Dash: Engineer C đang mở với last_seen_version=7 (đã lỗi thời)
    Dash->>WS: Publish sự kiện "feature_flag_x vừa được cập nhật lên version 8 bởi A"
    WS-->>EngineerC: Đẩy thông báo real-time lên màn hình đang mở

    EngineerC->>Dash: Thấy cảnh báo, chủ động tải lại config mới nhất trước khi sửa tiếp
    Dash->>FlagDB: SELECT value, version FROM CONFIG_ITEM WHERE id=feature_flag_x
    FlagDB-->>Dash: value mới nhất, version=8
    Dash-->>EngineerC: Cập nhật UI với version 8, Engineer C không còn submit dựa trên version 7 đã cũ
```

---

# Enhance sequence — Tra cứu audit log immutable để điều tra sự cố

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 4 của đề bài: mọi thay đổi config (dù qua optimistic lock thông thường hay đường khẩn cấp) đều được ghi vào `CONFIG_AUDIT_LOG` không thể xóa/sửa, phục vụ điều tra khi 1 thay đổi config gây ảnh hưởng không mong muốn.

```mermaid
sequenceDiagram
    actor Investigator as Kỹ sư điều tra sự cố
    participant Dash as Config Dashboard
    participant Log as CONFIG_AUDIT_LOG (immutable, append-only)

    Investigator->>Dash: Yêu cầu xem toàn bộ lịch sử thay đổi của feature_flag_x
    Dash->>Log: SELECT * FROM CONFIG_AUDIT_LOG WHERE config_item_id = feature_flag_x ORDER BY changed_at
    Log-->>Dash: Danh sách đầy đủ, gồm cả bản ghi is_emergency_override=true lúc buộc lưu
    Dash-->>Investigator: Hiển thị timeline, ai sửa, khi nào, giá trị trước/sau, có phải action khẩn cấp không

    Investigator->>Dash: Thử xóa/sửa 1 bản ghi audit log bị nghi ngờ (kiểm tra tính immutable)
    Dash->>Log: Kiểm tra quyền, hệ thống không cung cấp API xóa/sửa cho bảng CONFIG_AUDIT_LOG
    Log-->>Dash: Từ chối, audit log là append-only
    Dash-->>Investigator: Không thể xóa/sửa lịch sử, đảm bảo dữ liệu điều tra luôn đáng tin cậy
```
