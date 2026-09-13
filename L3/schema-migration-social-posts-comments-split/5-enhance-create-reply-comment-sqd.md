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
