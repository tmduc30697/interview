# Enhance sequence — Change member role (chính sách MFA áp dụng ngay, không chờ login lại)

Đây là **enhance** của flow `change-member-role` đã có ở base. So với base (đổi role không ảnh hưởng gì tới MFA), nay khi role thay đổi, hệ thống lập tức tra cứu lại `ORG_MFA_POLICY.applies_to_roles` cho role mới và áp dụng ngay lên session đang hoạt động, không chờ tới lần đăng nhập kế tiếp mới phát hiện ra chưa đủ điều kiện. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor Admin
    actor Viewer as User (đang là viewer, session active)
    participant Server
    participant DB as Database

    Note over Viewer: ORG_MFA_POLICY của Org1 chỉ bắt buộc MFA cho role admin/owner, viewer không cần

    Admin->>Server: PUT org member role (user=Viewer, role=admin)
    Server->>DB: UPDATE ORG_MEMBERSHIP SET role=admin WHERE user_id=Viewer, org_id=Org1
    DB-->>Server: OK

    Server->>DB: SELECT ORG_MFA_POLICY WHERE org_id=Org1
    DB-->>Server: required=true, applies_to_roles=[admin, owner]
    Server->>DB: SELECT MFA_ENROLLMENT WHERE user_id=Viewer
    DB-->>Server: không có enrollment nào

    Server->>DB: Đánh dấu SESSION hiện tại của Viewer cần re-auth kèm MFA ngay lập tức (không chờ session hết hạn)
    DB-->>Server: OK

    Server-->>Admin: Đổi role thành công

    Viewer->>Server: Request tiếp theo bất kỳ (vd mở 1 trang trong app)
    Server-->>Viewer: Chặn ngay, yêu cầu enroll và xác thực MFA trước khi tiếp tục vì role mới (admin) thuộc diện bắt buộc
```
