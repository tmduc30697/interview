# Sequence Diagram - Enhance: Follow / Unfollow

Đây là trạng thái **enhance** của flow "follow/unfollow" đã có ở base. So với base, flow này thay đổi ở: bảng `follows` nay duy trì index cho cả hai chiều `(follower_id, followee_id)` và `(followee_id, follower_id)` đánh đổi lấy chi phí ghi, đồng thời nếu người được follow đang ở diện fan-out/push thì follow/unfollow còn phải backfill hoặc dọn dẹp `feed_inbox` tương ứng.

```mermaid
sequenceDiagram
    actor U as User
    participant API as Social API
    participant DB as Postgres (follows)
    participant FanOut as Fan-out Worker

    U->>API: POST /follow/:targetId
    API->>DB: INSERT INTO follows(follower_id, followee_id)
    Note over DB: Ghi dong thoi duy tri 2 index (follower_id, followee_id) va (followee_id, follower_id)
    Note over DB: Toi uu ca hai chieu truy van, danh doi them chi phi ghi
    DB-->>API: OK
    alt targetId dang o dien fan-out/push
        API->>FanOut: backfill cac bai viet gan day cua targetId vao feed_inbox cua :me
        FanOut-->>API: da backfill xong
    end
    API-->>U: 200 da follow

    U->>API: DELETE /follow/:targetId
    API->>DB: DELETE FROM follows WHERE follower_id = :me AND followee_id = :targetId
    DB-->>API: OK
    alt targetId dang duoc fan-out cho :me
        API->>FanOut: xoa cac ban ghi cua targetId khoi feed_inbox cua :me
        FanOut-->>API: da don dep xong
    end
    API-->>U: 200 da unfollow

    U->>API: GET /followers (ai dang follow toi)
    API->>DB: SELECT follower_id FROM follows WHERE followee_id = :me
    Note over DB: Dung index (followee_id, follower_id) rieng cho chieu nay
    Note over DB: Nhanh hon dang ke so voi base khi chua co index toi uu cho chieu nay
    DB-->>API: danh sach followers
    API-->>U: tra ve danh sach
```
