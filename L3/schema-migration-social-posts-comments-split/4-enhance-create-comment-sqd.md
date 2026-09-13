# Sequence Diagram — Enhance: Create Comment

Đây là **enhance**, flow "bình luận vào bài viết" đã tồn tại ở base ([2-base-create-comment-sqd.md](2-base-create-comment-sqd.md)) nay thay đổi: bảng `COMMENT` insert bình thường không có vấn đề, nhưng ghi vào `comments_json` cũ phải lock dòng post trong lúc append để 2 comment gần như đồng thời trên 1 bài hot không ghi đè mất nhau.

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
