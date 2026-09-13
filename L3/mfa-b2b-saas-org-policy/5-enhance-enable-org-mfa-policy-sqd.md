# Enhance sequence — Enable org MFA policy (grace period có giới hạn + nhắc nhở tăng dần)

Đây là **enhance**, flow mới phát sinh từ đề bài — org admin bật chính sách MFA bắt buộc cho toàn tổ chức. Vì user đang có session hoạt động nhưng chưa enroll không thể bị ép logout tức thì (gây gián đoạn hàng loạt) nhưng cũng không thể dùng vô thời hạn, hệ thống tạo `GRACE_PERIOD_TRACKER` với hạn rõ ràng và nhắc nhở tăng dần. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Server
    participant DB as Database
    actor U as User chưa enroll MFA (đang có session active)

    Admin->>Server: PUT org MFA policy (required=true, grace_period_days=7)
    Server->>DB: UPSERT ORG_MFA_POLICY (org_id=Org1, required=true, grace_period_days=7, updated_by=Admin)
    DB-->>Server: OK

    Server->>DB: SELECT USER trong Org1 chưa có MFA_ENROLLMENT phù hợp
    DB-->>Server: danh sách gồm U
    Server->>DB: INSERT GRACE_PERIOD_TRACKER (user=U, org=Org1, deadline=now+7 ngày, reminder_count=0)
    DB-->>Server: OK

    Note over U: Session hiện tại của U vẫn tiếp tục hoạt động bình thường, không bị logout ngay

    Server-->>U: Thông báo trong app, "Tổ chức yêu cầu enroll MFA trước ngày X, còn 7 ngày"

    loop Mỗi ngày cho tới deadline
        Server->>DB: UPDATE GRACE_PERIOD_TRACKER SET reminder_count=reminder_count+1
        Server-->>U: Nhắc nhở, mức độ tăng dần theo số ngày còn lại, vd banner nhẹ → popup chặn thao tác → chỉ cho xem, không cho sửa
    end

    alt U enroll MFA trước deadline
        U->>Server: Hoàn tất enroll MFA
        Server->>DB: DELETE GRACE_PERIOD_TRACKER cho U
        Server-->>U: Tiếp tục dùng bình thường
    else U không enroll trước deadline
        Server->>DB: Đánh dấu session của U cần re-auth kèm MFA ở lần request tiếp theo
        Server-->>U: Chặn truy cập tính năng, yêu cầu enroll MFA ngay để tiếp tục
    end
```
