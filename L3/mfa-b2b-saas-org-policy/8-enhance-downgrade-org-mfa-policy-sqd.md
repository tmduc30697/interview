# Enhance sequence — Downgrade org MFA policy (tự yêu cầu MFA + log + thông báo admin khác)

Đây là **enhance**, flow mới phát sinh từ đề bài — org admin hạ cấp hoặc tắt chính sách MFA bắt buộc (vd đổi nhà cung cấp SSO). Vì đây là hành động làm giảm bảo mật toàn tổ chức, chính admin thực hiện phải tự xác thực MFA trước khi đổi được, đồng thời hệ thống ghi log và thông báo cho các admin khác trong tổ chức. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Admin as Org Admin (đang thực hiện thay đổi)
    participant Server
    participant DB as Database
    actor OtherAdmins as Các admin khác trong tổ chức

    Admin->>Server: PUT org MFA policy (required=false) để tắt bắt buộc MFA
    Server->>DB: SELECT ORG_MFA_POLICY hiện tại WHERE org_id=Org1
    DB-->>Server: required=true (đang bật)
    Note over Server: Phát hiện đây là hành động hạ cấp bảo mật (true → false)

    Server-->>Admin: Yêu cầu xác thực lại MFA để xác nhận thay đổi này
    Admin->>Server: Nhập mã MFA hiện tại

    alt Xác thực MFA thành công
        Server->>DB: UPDATE ORG_MFA_POLICY SET required=false, updated_by=Admin, updated_at=now()
        Server->>DB: INSERT MFA_POLICY_AUDIT_LOG (org=Org1, action=disable, performed_by=Admin)
        DB-->>Server: OK
        Server->>OtherAdmins: Gửi thông báo, "Admin X vừa tắt chính sách MFA bắt buộc của tổ chức lúc <thời điểm>"
        Server-->>Admin: Thay đổi thành công
    else Xác thực MFA thất bại
        Server-->>Admin: Từ chối thay đổi, chính sách giữ nguyên required=true
    end
```
