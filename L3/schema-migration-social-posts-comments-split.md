# Tách comment dạng nested JSON trong bảng posts sang bảng comment quan hệ riêng (mạng xã hội)

**Hệ thống:** Một mạng xã hội có bảng `posts` lưu comment dạng nested/JSON ngay trong 1 cột của bài viết, cần tách sang bảng `comments` quan hệ riêng để hỗ trợ tính năng mới (trả lời lồng nhiều cấp, like riêng từng comment, phân trang hiệu quả), trong khi traffic đăng bài/bình luận vẫn chạy liên tục 24/7.

**Vai trò của flow:** Flow dual-write phải đảm bảo mọi comment mới được ghi cả vào JSON cũ và bảng `comments` mới, có job backfill comment lịch sử, mà không làm rớt comment hay đổi thứ tự hiển thị trong lúc traffic bình luận vẫn cao liên tục.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: 2 người dùng cùng bình luận vào 1 bài viết đang hot gần như đồng thời (chênh nhau vài trăm mili giây) trong giai đoạn dual-write — việc ghi vào cột JSON cũ (thường phải đọc JSON hiện tại, append, rồi ghi lại cả mảng) không được dùng kiểu đọc-sửa-ghi không atomic vì 2 request đọc cùng JSON gốc rồi ghi đè nhau sẽ làm mất 1 trong 2 comment; yêu cầu mô tả cụ thể cách xử lý (lock dòng post trong lúc append JSON, hoặc chuyển hẳn logic ghi JSON sang dạng append atomic ở tầng lưu trữ), trong khi bảng `comments` mới insert bình thường không gặp vấn đề này vì mỗi comment là 1 dòng riêng.
- Trong giai đoạn đọc vẫn lấy từ JSON cũ (nguồn sự thật) nhưng ghi đã chạy song song cả 2 nơi, phải đảm bảo thứ tự comment trong JSON và thứ tự theo timestamp trong bảng `comments` khớp nhau tuyệt đối; yêu cầu cơ chế kiểm tra để phát hiện lệch thứ tự (ví dụ do 1 trong 2 lần ghi bị chậm hơn lần kia dưới tải cao) trước khi cho phép chuyển đọc sang bảng mới.
- Mô tả cụ thể race với reply lồng nhiều cấp: 1 comment cha vừa được tạo và ngay lập tức có người trả lời comment đó (comment con) trước khi thao tác ghi JSON cho comment cha kịp hoàn tất ở phía JSON cũ (do append JSON có độ trễ cao hơn insert bảng quan hệ) — quy định rõ cách xử lý để comment con không bị ghi vào JSON cũ ở vị trí sai cấu trúc lồng, ví dụ chờ xác nhận comment cha đã ghi xong JSON trước khi cho phép reply được xử lý, hoặc retry ghi JSON theo đúng thứ tự cha rồi mới tới con.
- Job backfill comment lịch sử phải chạy theo batch ưu tiên các bài viết có traffic bình luận đang hoạt động thấp trước, có checkpoint để resume nếu crash giữa chừng, và phải bỏ qua/log riêng các bài viết có comment JSON bị lỗi cấu trúc (dữ liệu cũ có thể không đồng nhất do đã đổi format nhiều lần trước đây) thay vì làm crash toàn bộ job.
- Có job reconciliation so sánh số lượng comment và nội dung mẫu giữa JSON và bảng mới cho các bài viết có traffic cao mỗi ngày, và chỉ được drop cột JSON cũ sau khi 100% traffic đọc đã chuyển sang bảng mới và không phát hiện lệch trong khoảng thời gian tối thiểu quy định.

---

# ERD — Base (trước khi tách bảng comments)

Đây là **base**: mô hình dữ liệu suy luận cho mạng xã hội *trước khi* tách comment. Đề bài giả định đã có `USER` đăng `POST`, và comment được lưu nested/JSON ngay trong 1 cột của `POST` — nếu không có sẵn cột JSON đó thì yêu cầu "tách sang bảng comments quan hệ riêng" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ đăng bài/bình luận, không suy diễn thêm like/share/notification không liên quan.

```mermaid
erDiagram
    USER ||--o{ POST : creates

    USER {
        string user_id PK
        string username
        string email
    }

    POST {
        string post_id PK
        string user_id FK
        string content
        string comments_json
        datetime created_at
    }
```

---

<a id="2-base-create-comment-sqd"></a>

# Sequence Diagram — Base: Create Comment

Đây là **base**, flow "bình luận vào bài viết" — tiền đề bắt buộc cho việc tách bảng: chính flow này đọc mảng JSON hiện tại, append comment mới, rồi ghi lại cả mảng vào cột `comments_json` của `POST` — cách đọc-sửa-ghi này là nguồn gốc rủi ro mất comment mà enhance phải xử lý.

```mermaid
sequenceDiagram
    actor User
    participant CommentSvc as Comment Service
    participant DB as Posts Table

    User->>CommentSvc: Submit comment on post
    CommentSvc->>DB: SELECT comments_json FROM posts WHERE post_id = ?
    DB-->>CommentSvc: Current comments array
    CommentSvc->>CommentSvc: Append new comment to array
    CommentSvc->>DB: UPDATE posts SET comments_json = updated array
    DB-->>CommentSvc: Updated
    CommentSvc-->>User: Comment posted

    Note over CommentSvc,DB: Đọc-sửa-ghi không atomic, 2 request đọc cùng JSON gốc rồi ghi đè nhau sẽ làm mất 1 comment
```

---

# ERD — Enhance (sau khi tách bảng comments)

Đây là **enhance**: ERD base cộng với bảng `COMMENT` quan hệ, tự tham chiếu để hỗ trợ trả lời lồng nhiều cấp, checkpoint backfill theo bài viết, và báo cáo đối soát thứ tự/nội dung. So với base, `POST` giữ nguyên `comments_json` (chưa drop, vẫn dual-write) trong khi `COMMENT` là bảng hoàn toàn mới cho phép mỗi comment là 1 dòng riêng, có `parent_comment_id` tự tham chiếu để biểu diễn reply lồng cấp — điều JSON cũ không hỗ trợ tốt.

```mermaid
erDiagram
    USER ||--o{ POST : creates
    USER ||--o{ COMMENT : writes
    POST ||--o{ COMMENT : "has (new, dual-written)"
    COMMENT ||--o{ COMMENT : "replies to (self-referencing)"
    POST ||--o| ORDER_CONSISTENCY_CHECK : "validated by"

    USER {
        string user_id PK
        string username
        string email
    }

    POST {
        string post_id PK
        string user_id FK
        string content
        string comments_json
        datetime created_at
    }

    COMMENT {
        string comment_id PK
        string post_id FK
        string parent_comment_id FK
        string user_id FK
        string content
        datetime created_at
    }

    BACKFILL_CHECKPOINT {
        string checkpoint_id PK
        string last_post_id_processed
        string status
        datetime updated_at
    }

    ORDER_CONSISTENCY_CHECK {
        string check_id PK
        string post_id FK
        boolean order_matches
        datetime checked_at
    }
```

---

# Sequence Diagram — Enhance: Create Comment

Đây là **enhance**, flow "bình luận vào bài viết" đã tồn tại ở base ([2-base-create-comment-sqd.md](#2-base-create-comment-sqd)) nay thay đổi: bảng `COMMENT` insert bình thường không có vấn đề, nhưng ghi vào `comments_json` cũ phải lock dòng post trong lúc append để 2 comment gần như đồng thời trên 1 bài hot không ghi đè mất nhau.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    participant CommentSvc as Comment Service
    participant DB as Posts + Comments Tables

    par 2 comment gần như đồng thời trên cùng 1 bài viết hot
        UserA->>CommentSvc: Submit comment A
    and
        UserB->>CommentSvc: Submit comment B
    end

    CommentSvc->>DB: INSERT INTO comments (post_id, content) for comment A
    DB-->>CommentSvc: comment A inserted, không có vấn đề gì

    CommentSvc->>DB: Lock post row (post_id) for update, append comment A vào comments_json
    DB-->>CommentSvc: Lock acquired, JSON updated với comment A
    CommentSvc->>DB: Release lock

    CommentSvc->>DB: INSERT INTO comments (post_id, content) for comment B
    DB-->>CommentSvc: comment B inserted, không có vấn đề gì

    CommentSvc->>DB: Lock post row (post_id) for update, append comment B vào comments_json
    DB-->>CommentSvc: Lock acquired, chờ lock A giải phóng trước, JSON updated với comment B
    CommentSvc->>DB: Release lock

    CommentSvc-->>UserA: Comment A posted
    CommentSvc-->>UserB: Comment B posted

    Note over DB: Lock dòng post khi append JSON đảm bảo không có 2 lần đọc-sửa-ghi chồng lên nhau làm mất comment
```

---

# Sequence Diagram — Enhance: Create Reply Comment

Đây là **enhance**, flow hoàn toàn mới: bình luận trả lời 1 comment cha, cần chờ comment cha ghi xong JSON cũ trước khi cho phép xử lý reply, để không bị ghi vào `comments_json` sai vị trí cấu trúc lồng do append JSON có độ trễ cao hơn insert bảng `COMMENT`.

```mermaid
sequenceDiagram
    actor Parent as User (comment cha)
    actor Replier as User (trả lời ngay sau đó)
    participant CommentSvc as Comment Service
    participant DB as Posts + Comments Tables

    Parent->>CommentSvc: Submit comment cha
    CommentSvc->>DB: INSERT INTO comments (parent_comment_id = null) for comment cha
    DB-->>CommentSvc: Comment cha inserted (bảng quan hệ xong ngay)
    CommentSvc-->>Parent: Comment cha posted

    par CommentSvc tiếp tục ghi JSON cho comment cha (chậm hơn)
        CommentSvc->>DB: Lock post row, append comment cha vào comments_json
    and Replier gửi reply gần như ngay lập tức
        Replier->>CommentSvc: Submit reply cho comment cha
    end

    CommentSvc->>DB: INSERT INTO comments (parent_comment_id = comment cha id) for reply
    DB-->>CommentSvc: Reply inserted vào bảng quan hệ (không phụ thuộc JSON)

    CommentSvc->>CommentSvc: Kiểm tra comment cha đã ghi xong JSON chưa
    alt Comment cha đã ghi xong JSON
        CommentSvc->>DB: Append reply vào đúng vị trí lồng trong comments_json
    else Comment cha chưa ghi xong JSON
        CommentSvc->>CommentSvc: Xếp hàng đợi, retry ghi JSON cho reply sau khi comment cha ghi xong
        CommentSvc->>DB: Append comment cha vào comments_json (hoàn tất)
        CommentSvc->>DB: Append reply vào đúng vị trí con của comment cha trong comments_json
    end

    CommentSvc-->>Replier: Reply posted
```

---

# Sequence Diagram — Enhance: Backfill Historical Comments

Đây là **enhance**, flow hoàn toàn mới: job backfill comment lịch sử từ `comments_json` sang bảng `COMMENT`, ưu tiên bài viết ít traffic bình luận trước, có checkpoint resume, và bỏ qua riêng các bài có JSON lỗi cấu trúc thay vì crash toàn bộ job.

```mermaid
sequenceDiagram
    participant Job as Backfill Job
    participant Checkpoint as Backfill Checkpoint Store
    participant DB as Posts + Comments Tables

    Job->>Checkpoint: Read last_post_id_processed
    Checkpoint-->>Job: Resume cursor

    loop Until no more posts
        Job->>DB: SELECT next posts ORDER BY current_comment_traffic ASC, post_id > cursor
        DB-->>Job: Batch of posts with comments_json

        loop for each post in batch
            Job->>Job: Parse comments_json (bao gồm cấu trúc lồng nhiều cấp)
            alt JSON hợp lệ
                Job->>DB: INSERT INTO comments (post_id, parent_comment_id, content, created_at) cho từng comment theo đúng thứ tự cha-con
            else JSON lỗi cấu trúc, do đổi format nhiều lần trước đây
                Job->>Job: Log riêng post_id bị lỗi, bỏ qua post này
            end
        end

        Job->>Checkpoint: Update last_post_id_processed
        Checkpoint-->>Job: Checkpoint saved
    end

    Job-->>Job: Backfill completed, danh sách post lỗi cấu trúc được log riêng để xử lý thủ công
```
