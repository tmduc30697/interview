# Sequence Diagram - Enhance: Batch job tính engagement score

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base. Flow này giải quyết vấn đề đề bài nêu: khi feed cần xếp hạng theo mức độ liên quan, điểm số biến động liên tục theo like/comment mới khiến B-tree index không tối ưu để sort real-time, nên cần một batch job định kỳ pre-computed `engagement_score` thay vì tính lúc đọc feed.

```mermaid
sequenceDiagram
    participant Scheduler as Cron Scheduler
    participant Job as Engagement Score Job
    participant DB as Postgres (posts, likes, comments)

    Scheduler->>Job: kich hoat dinh ky (vi du moi 5-15 phut)
    Job->>DB: dem so luot like/comment moi cho tung bai viet gan day
    DB-->>Job: so lieu like/comment
    Job->>Job: tinh engagement_score (vi du trong so theo thoi gian dang, so luot like, comment)
    Job->>DB: UPDATE posts SET engagement_score = :score WHERE id = :post_id
    Note over DB: engagement_score duoc luu san (materialized), khong tinh real-time luc doc feed
    DB-->>Job: da cap nhat xong
    Note over Job: Flow xem newsfeed sau nay chi can ORDER BY engagement_score co san, khong phai JOIN/tinh toan like-comment ngay luc user xem feed
```
