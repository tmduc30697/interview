# Sequence Diagram - Base: Follow / Unfollow

Đây là trạng thái **base**, flow "follow/unfollow" và truy vấn bảng `follows` theo cả hai chiều (tôi follow ai, ai follow tôi) khi chưa quyết định thứ tự cột index. Flow này được chọn vì đề bài đặt thẳng câu hỏi nên đánh index `(follower_id, followee_id)` hay ngược lại, và base cần cho thấy vấn đề khi chỉ có (hoặc chưa có) index tối ưu cho một chiều.

```mermaid
sequenceDiagram
    actor U as User
    participant API as Social API
    participant DB as Postgres (follows)

    U->>API: POST /follow/:targetId
    API->>DB: INSERT INTO follows(follower_id, followee_id) VALUES (:me, :targetId)
    DB-->>API: OK
    API-->>U: 200 da follow

    U->>API: GET /following (nhung nguoi toi dang follow)
    API->>DB: SELECT followee_id FROM follows WHERE follower_id = :me
    DB-->>API: danh sach following
    API-->>U: tra ve danh sach following

    U->>API: GET /followers (ai dang follow toi)
    API->>DB: SELECT follower_id FROM follows WHERE followee_id = :me
    Note over DB: Chua ro index nao toi uu cho chieu truy van theo followee_id
    Note over DB: Co the phai quet toan bang neu bang follows lon
    DB-->>API: danh sach followers (co the cham)
    API-->>U: tra ve danh sach followers

    U->>API: DELETE /follow/:targetId
    API->>DB: DELETE FROM follows WHERE follower_id = :me AND followee_id = :targetId
    DB-->>API: OK
    API-->>U: 200 da unfollow
```
