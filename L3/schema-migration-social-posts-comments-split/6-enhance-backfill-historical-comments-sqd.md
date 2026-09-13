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
