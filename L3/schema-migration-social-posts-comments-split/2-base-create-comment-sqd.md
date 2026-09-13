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
