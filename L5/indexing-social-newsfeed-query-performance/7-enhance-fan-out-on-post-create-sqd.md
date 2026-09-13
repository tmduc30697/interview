# Sequence Diagram - Enhance: Đăng bài và fan-out ghi feed

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base. Flow này thể hiện ngưỡng chuyển từ pull-query sang mô hình fan-out/push mà đề bài đặt ra: khi số follower dưới ngưỡng, hệ thống ghi sẵn bài viết vào `feed_inbox` của từng follower ngay lúc đăng bài (đánh đổi write amplification để đọc feed nhanh); khi tác giả là celebrity có quá nhiều follower, bỏ qua fan-out ghi trước và để pull-query xử lý lúc đọc (mô hình hybrid).

```mermaid
sequenceDiagram
    actor U as Author
    participant API as Post API
    participant DB as Postgres (posts)
    participant FanOut as Fan-out Worker
    participant Inbox as Postgres (feed_inbox)

    U->>API: POST /posts (dang bai viet moi)
    API->>DB: INSERT INTO posts(author_id, content, created_at)
    DB-->>API: post_id
    API-->>U: 201 da dang bai

    API->>FanOut: enqueue fan-out job cho post_id
    FanOut->>DB: dem so luong follower cua author_id
    DB-->>FanOut: so luong follower

    alt so follower duoi nguong fan-out
        FanOut->>Inbox: INSERT feed_inbox(owner_id, post_id) cho tung follower
        Inbox-->>FanOut: da ghi xong
    else tac gia la celebrity, vuot xa nguong
        Note over FanOut: Bo qua fan-out ghi truoc de tranh write amplification qua lon
        Note over FanOut: Follower cua celebrity se doc bai nay qua pull-query luc xem feed
    end
```
