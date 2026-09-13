# Chống spam tin nhắn trong app chat/group

**Hệ thống:** Ứng dụng chat/group cho phép user gửi tin nhắn tự do, cần chặn bot/spam gửi hàng loạt tin nhắn mà không làm phiền user thật đang gõ nhanh trong cuộc trò chuyện sôi nổi.

**Vai trò của flow:** Rate limiting theo user/theo cuộc trò chuyện giới hạn tần suất gửi tin để phát hiện và chặn hành vi spam, đồng thời đủ khoan dung cho tốc độ gõ/gửi tự nhiên của người dùng bình thường.

**Yêu cầu cụ thể:**
- Ngưỡng rate limit cứng (ví dụ giới hạn tin/giây cố định) dễ chặn nhầm user thật đang nhắn tin dồn dập trong lúc trò chuyện sôi nổi — cần thuật toán cho phép burst ngắn tự nhiên (vài tin liên tiếp trong 1-2 giây) nhưng vẫn bắt được pattern đều đặn máy móc kéo dài của bot.
- Phân biệt rate limit theo từng cuộc trò chuyện một-một và theo group lớn (hàng trăm/nghìn thành viên) — cùng một user gửi nhanh trong group đông có thể là bình thường (đang thảo luận sôi nổi), trong khi cùng tần suất đó nhắm vào nhiều user khác nhau (spam hàng loạt) phải bị đánh dấu khác.
- Khi user bị giới hạn do gửi quá nhanh, phản hồi (ẩn tạm nút gửi, thông báo chờ vài giây) không được làm mất nội dung tin nhắn đang gõ dở — user không được yêu cầu gõ lại từ đầu chỉ vì bị rate limit tạm thời.
- Phát hiện spam không chỉ dựa vào tần suất mà còn pattern nội dung lặp lại (cùng một tin gửi liên tục tới nhiều group/user khác nhau trong thời gian ngắn) — một tài khoản gửi đúng nhịp một tin/giây liên tục hàng giờ tới nhiều đối tượng khác nhau là dấu hiệu bot dù tần suất từng tin không vượt ngưỡng cứng.
- Rate limit phải áp dụng được động theo uy tín tài khoản (tài khoản mới tạo/chưa xác thực bị giới hạn chặt hơn tài khoản lâu năm có lịch sử hành vi tốt), tránh chặn quá nghiêm với user thật lâu năm trong khi vẫn kiểm soát chặt tài khoản mới dễ bị lợi dụng để spam.

---

# Base ERD — Chat/group với rate limit ngưỡng cứng cố định

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có thuật toán cho phép burst tự nhiên, phân biệt theo cuộc trò chuyện, phát hiện pattern nội dung, và điều chỉnh theo uy tín tài khoản. Đề bài nói tới ngưỡng rate limit cứng (giới hạn tin/giây cố định) đang gây chặn nhầm user thật — nên base cần đủ: user, cuộc trò chuyện (1-1 hoặc group), tin nhắn, và 1 bộ đếm rate limit đơn giản theo cửa sổ thời gian cố định áp dụng chung cho mọi user, không phân biệt loại cuộc trò chuyện hay uy tín tài khoản. Base **chưa có** token bucket cho phép burst, chưa có policy theo trust level, chưa có phát hiện pattern nội dung lặp lại — những thứ đó là phần enhance.

```mermaid
erDiagram
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ CONVERSATION_MEMBER : has
    USER ||--o{ CONVERSATION_MEMBER : "is member of"
    USER ||--o{ RATE_LIMIT_COUNTER : "tracked by"

    USER {
        string id PK
        string name
        datetime created_at
        boolean is_verified
    }
    CONVERSATION {
        string id PK
        string type "direct | group"
    }
    CONVERSATION_MEMBER {
        string conversation_id FK
        string user_id FK
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string content
        datetime sent_at
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string user_id FK
        datetime window_start
        int message_count "reset mỗi cửa sổ cố định, vd mỗi giây"
    }
```

---

# Base sequence — Gửi tin nhắn với rate limit ngưỡng cứng, chặn nhầm khi gõ dồn dập

Đây là **base**, flow "Gửi tin nhắn" ở trạng thái hiện tại — mỗi lần gửi, app cộng dồn bộ đếm theo cửa sổ thời gian cố định (vd 1 giây) áp dụng chung cho mọi user, không phân biệt gửi trong group đông hay chat 1-1, vượt ngưỡng là chặn ngay và mất luôn nội dung đang gõ. Flow này liên quan mật thiết tới enhance vì toàn bộ 4 yêu cầu của đề bài đều nhằm sửa đúng các hạn chế ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat Service
    participant DB as RATE_LIMIT_COUNTER + MESSAGE store

    User->>App: Gửi tin nhắn 1 trong group đang thảo luận sôi nổi
    App->>DB: Đọc RATE_LIMIT_COUNTER(user), message_count=1 trong cửa sổ hiện tại
    App->>DB: Ghi MESSAGE, tăng message_count=2
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 2 ngay sau đó (vẫn đang trò chuyện sôi nổi, cách 0.5 giây)
    App->>DB: message_count=3, vẫn dưới ngưỡng cứng 3 tin/giây
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 3 (cách 0.3 giây, vẫn là người thật gõ nhanh)
    App->>DB: message_count=4, vượt ngưỡng cứng 3 tin/giây
    App-->>User: Từ chối gửi, ẩn nút gửi, xoá luôn nội dung đang gõ trong ô nhập liệu

    Note over App,DB: Ngưỡng cố định không phân biệt burst tự nhiên của user thật đang sôi nổi trong group đông với pattern đều đặn của bot, và làm mất nội dung tin nhắn khi chặn
```

---

# Enhance ERD — Token bucket theo scope, policy theo trust level, phát hiện pattern nội dung

Đây là **enhance**, mô hình dữ liệu sau khi áp thuật toán rate limit cho phép burst + phân biệt theo cuộc trò chuyện + phát hiện pattern nội dung + policy động theo uy tín tài khoản lên base. So với base: `RATE_LIMIT_COUNTER` (cửa sổ cố định) được thay bằng `RATE_LIMIT_BUCKET` (token bucket cho phép burst ngắn, tách theo `scope` — trong 1 conversation cụ thể hay tổng hợp toàn user). `USER` thêm `trust_level` (suy ra từ tuổi tài khoản + lịch sử hành vi), gắn với `RATE_LIMIT_POLICY` quy định `burst_capacity`/`sustained_rate` khác nhau theo `trust_level` và loại conversation. Thêm mới `CONTENT_SPAM_SIGNAL` — theo dõi nội dung lặp lại gửi tới nhiều đối tượng khác nhau trong thời gian ngắn, phát hiện bot dù tần suất từng tin không vượt ngưỡng cứng.

```mermaid
erDiagram
    USER ||--o{ MESSAGE : sends
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ CONVERSATION_MEMBER : has
    USER ||--o{ CONVERSATION_MEMBER : "is member of"
    USER ||--o{ RATE_LIMIT_BUCKET : "throttled by"
    USER ||--|| RATE_LIMIT_POLICY : "governed by (theo trust_level hiện tại)"
    USER ||--o{ CONTENT_SPAM_SIGNAL : "flagged by"

    USER {
        string id PK
        string name
        datetime created_at
        boolean is_verified
        string trust_level "new | standard | trusted"
    }
    CONVERSATION {
        string id PK
        string type "direct | group"
        int member_count
    }
    CONVERSATION_MEMBER {
        string conversation_id FK
        string user_id FK
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string content
        string content_hash "dùng để so khớp nội dung lặp lại"
        datetime sent_at
    }
    RATE_LIMIT_BUCKET {
        string id PK
        string user_id FK
        string scope "per_conversation:<id> | per_user_aggregate"
        decimal tokens_remaining
        decimal bucket_capacity "burst tối đa cho phép"
        decimal refill_rate_per_sec "tốc độ nạp lại token, quyết định tần suất bền vững"
        datetime last_refill_at
    }
    RATE_LIMIT_POLICY {
        string id PK
        string trust_level "new | standard | trusted"
        string conversation_type "direct | group"
        decimal burst_capacity
        decimal sustained_rate_per_sec
    }
    CONTENT_SPAM_SIGNAL {
        string id PK
        string user_id FK
        string content_hash
        int distinct_target_count "số conversation/user khác nhau nhận cùng nội dung"
        datetime first_seen_at
        datetime last_seen_at
        string pattern_detected "regular_interval_bot | none"
    }
```

---

# Enhance sequence — Gửi tin nhắn với token bucket cho phép burst, giữ nguyên nội dung khi bị chặn tạm

Đây là **enhance**, flow "Gửi tin nhắn" sau khi áp token bucket theo scope. So với base (ngưỡng cứng, mất nội dung khi chặn), flow này thay đổi ở 3 điểm: (1) dùng token bucket cho phép burst ngắn (vài tin liên tiếp trong 1-2 giây) miễn còn token, chỉ bắt pattern đều đặn kéo dài khi bucket cạn và refill_rate không theo kịp, (2) scope tách theo conversation (`per_conversation:<id>`) để gửi nhanh trong 1 group đông không tính chung với gửi rải rác tới nhiều người khác, (3) khi bị chặn tạm, chỉ ẩn nút gửi + đếm ngược vài giây, nội dung đang gõ vẫn được giữ nguyên trong ô nhập liệu.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat Service
    participant DB as RATE_LIMIT_BUCKET + RATE_LIMIT_POLICY store

    User->>App: Gửi tin nhắn 1 trong group đông đang thảo luận sôi nổi
    App->>DB: Đọc RATE_LIMIT_BUCKET(user, scope=per_conversation:group-42), tokens_remaining=3/3
    App->>DB: Trừ 1 token, tokens_remaining=2
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 2 (cách 0.5 giây)
    App->>DB: tokens_remaining=2, đủ token, trừ còn 1
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 3 (cách 0.3 giây, vẫn trong burst ngắn tự nhiên)
    App->>DB: tokens_remaining=1, đủ token, trừ còn 0
    App-->>User: Gửi thành công

    User->>App: Gửi tin nhắn 4 ngay lập tức (bucket đã cạn, refill_rate chưa kịp nạp lại)
    App->>DB: tokens_remaining=0, không đủ token

    App-->>User: Tạm ẩn nút gửi, hiện đếm ngược "chờ 1 giây", giữ nguyên nội dung tin nhắn 4 trong ô nhập liệu

    Note over DB: Sau đúng thời gian refill_rate quy định, bucket có lại token, user chỉ cần bấm gửi lại đúng nội dung đã gõ sẵn, không phải gõ lại từ đầu

    Note over App,DB: Nếu cùng nhịp gửi đều đặn này lặp lại liên tục hàng giờ (không có khoảng nghỉ tự nhiên của người thật), pattern đều đặn máy móc sẽ bị phát hiện ở flow 5-enhance-content-pattern-detection-sqd.md dù từng tin không vượt token bucket
```

---

# Enhance sequence — Phát hiện bot qua nội dung lặp lại gửi tới nhiều đối tượng khác nhau

Đây là **enhance**, flow mới phát sinh từ enhance — phát hiện spam không chỉ dựa vào tần suất (vốn có thể luôn nằm dưới ngưỡng token bucket) mà còn dựa vào pattern nội dung: cùng 1 tin gửi liên tục tới nhiều group/user khác nhau trong thời gian ngắn với nhịp đều đặn máy móc. Đúng yêu cầu thứ 4 của đề bài — 1 tài khoản gửi đúng nhịp 1 tin/giây liên tục hàng giờ tới nhiều đối tượng khác nhau là dấu hiệu bot dù từng tin không vượt ngưỡng cứng.

```mermaid
sequenceDiagram
    actor Bot as Tài khoản bot
    participant App as Chat Service
    participant DB as MESSAGE + RATE_LIMIT_BUCKET store
    participant Detector as Content Spam Detector (background)
    participant Signal as CONTENT_SPAM_SIGNAL store
    participant Mod as Đội kiểm duyệt/hệ thống chặn tự động

    loop Mỗi giây, liên tục hàng giờ
        Bot->>App: Gửi cùng 1 nội dung tới 1 user/group khác nhau mỗi lần
        App->>DB: Kiểm tra RATE_LIMIT_BUCKET, luôn đủ token vì nhịp đúng 1 tin/giây không vượt burst
        App->>DB: Ghi MESSAGE(content_hash=giống nhau mỗi lần)
        App-->>Bot: Gửi thành công
    end

    par Detector chạy song song, quét theo content_hash
        Detector->>DB: Gom nhóm MESSAGE theo (sender_id, content_hash) trong cửa sổ thời gian gần đây
        Detector->>Detector: Tính distinct_target_count (số conversation/user khác nhau nhận cùng nội dung) và độ đều đặn của khoảng cách thời gian giữa các lần gửi
        Detector->>Detector: distinct_target_count cao + khoảng cách gần như cố định (đều đặn máy móc) => pattern_detected=regular_interval_bot
        Detector->>Signal: Ghi CONTENT_SPAM_SIGNAL(pattern_detected=regular_interval_bot)
    end

    Signal-->>Mod: Cảnh báo tài khoản nghi bot dù chưa từng vượt token bucket
    Mod->>App: Áp policy nghiêm ngặt hơn hoặc tạm khoá tài khoản để xác minh

    Note over Detector,Signal: Cơ chế này bắt được đúng trường hợp tần suất từng tin luôn hợp lệ nhưng tổng thể hành vi là bot rải tin hàng loạt
```

---

# Enhance sequence — Rate limit động theo uy tín tài khoản

Đây là **enhance**, flow mới phát sinh từ enhance — chọn `RATE_LIMIT_POLICY` áp dụng cho token bucket của mỗi user dựa trên `trust_level`, đảm bảo tài khoản mới/chưa xác thực bị giới hạn chặt hơn trong khi tài khoản lâu năm có lịch sử hành vi tốt không bị chặn quá nghiêm. Đúng yêu cầu cuối của đề bài.

```mermaid
sequenceDiagram
    actor NewUser as User mới tạo tài khoản
    actor OldUser as User lâu năm, hành vi tốt
    participant App as Chat Service
    participant Policy as RATE_LIMIT_POLICY store
    participant Bucket as RATE_LIMIT_BUCKET store
    participant Trust as Trust Level Evaluator (background)

    par Đánh giá trust_level định kỳ cho mọi user
        Trust->>Trust: Tính trust_level dựa trên tuổi tài khoản, is_verified, lịch sử bị flag/report
        Trust->>App: UPDATE USER SET trust_level=new (tài khoản mới, chưa xác thực)
        Trust->>App: UPDATE USER SET trust_level=trusted (tài khoản lâu năm, không có report xấu)
    end

    NewUser->>App: Gửi tin nhắn trong group
    App->>Policy: Tra RATE_LIMIT_POLICY(trust_level=new, conversation_type=group)
    Policy-->>App: burst_capacity=2, sustained_rate=0.5 tin/giây (chặt)
    App->>Bucket: Áp dụng policy chặt lên RATE_LIMIT_BUCKET của NewUser
    App-->>NewUser: Gửi thành công, nhưng burst tối đa chỉ 2 tin liên tiếp

    OldUser->>App: Gửi tin nhắn trong group
    App->>Policy: Tra RATE_LIMIT_POLICY(trust_level=trusted, conversation_type=group)
    Policy-->>App: burst_capacity=8, sustained_rate=3 tin/giây (khoan dung)
    App->>Bucket: Áp dụng policy khoan dung lên RATE_LIMIT_BUCKET của OldUser
    App-->>OldUser: Gửi thành công, burst tới 8 tin liên tiếp không bị chặn

    Note over Policy,Bucket: Cùng 1 hành vi gõ dồn dập nhưng tài khoản mới bị giới hạn chặt hơn để kiểm soát rủi ro spam, tài khoản lâu năm được khoan dung hơn nhờ lịch sử hành vi tốt
```
