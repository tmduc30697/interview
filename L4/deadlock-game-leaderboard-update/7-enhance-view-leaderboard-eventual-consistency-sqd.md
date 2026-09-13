# Enhance sequence — Xem leaderboard (chấp nhận rank trễ vài phút, đổi lấy UX nhanh)

Đây là **enhance**, flow mới minh họa trade-off giữa isolation level nhanh cho transaction cộng điểm và độ trễ chấp nhận được của rank. Đáp ứng yêu cầu 3 của đề bài: `READ COMMITTED` cho transaction cộng điểm giúp UX nhanh, nhưng rank hiển thị cho user có thể trễ tới khi job batch tiếp theo chạy xong, không cần chính xác tuyệt đối theo thời gian thực.

```mermaid
sequenceDiagram
    actor User
    participant App as Learning App
    participant DB as Database
    participant Score as USER_SCORE (user X)

    User->>App: Hoàn thành bài học lúc 10:00:05
    App->>DB: Transaction cộng điểm (READ COMMITTED), COMMIT ngay
    DB->>Score: total_score = 120 (cộng thêm điểm mới)
    Note over Score: rank vẫn giữ giá trị cũ, chưa được tính lại

    User->>App: Mở màn hình leaderboard lúc 10:00:10
    App->>DB: Đọc USER_SCORE (user X)
    DB-->>App: total_score=120, rank=5 (rank_updated_at=09:57, trước khi job batch tiếp theo chạy)
    App-->>User: Hiển thị điểm mới nhất (120) nhưng rank vẫn là 5 (có thể chưa phản ánh đúng vị trí thực tế)

    Note over App,DB: Ví dụ cụ thể: user X vừa vượt qua user Y về điểm số, nhưng rank hiển thị vẫn xếp X sau Y cho tới khi Rank Job batch kế tiếp chạy (vài phút sau) và cập nhật lại rank_updated_at, đây là đánh đổi chấp nhận được vì rank không phải dữ liệu cần chính xác tức thời như điểm số
```
