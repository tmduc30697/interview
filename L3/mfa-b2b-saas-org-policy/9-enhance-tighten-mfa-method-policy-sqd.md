# Enhance sequence — Tighten MFA method policy (buộc enroll lại phương thức mới, có thời hạn)

Đây là **enhance**, flow mới phát sinh từ đề bài — org admin siết chính sách sang chỉ cho phép 1 phương thức MFA chặt hơn (vd chỉ WebAuthn, cấm SMS OTP). Những user đã enroll SMS theo chính sách cũ được cấp thời hạn rõ ràng để enroll lại, không bị mất quyền truy cập ngay khi chính sách vừa đổi. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Server
    participant DB as Database
    actor U as User đã enroll SMS OTP

    Admin->>Server: PUT org MFA policy (allowed_methods=[webauthn], cấm sms)
    Server->>DB: UPDATE ORG_MFA_POLICY SET allowed_methods=[webauthn] WHERE org_id=Org1
    DB-->>Server: OK
    Server->>DB: INSERT MFA_POLICY_AUDIT_LOG (org=Org1, action=tighten_method, performed_by=Admin)

    Server->>DB: SELECT MFA_ENROLLMENT WHERE org member thuộc Org1, method NOT IN allowed_methods
    DB-->>Server: danh sách gồm U (method=sms)
    Server->>DB: UPDATE MFA_ENROLLMENT SET must_reenroll_by=now()+14 ngày WHERE user=U
    DB-->>Server: OK

    Server-->>Admin: Cập nhật chính sách thành công

    U->>Server: Đăng nhập bằng SMS OTP (vẫn còn trong hạn 14 ngày)
    Server->>DB: Kiểm tra must_reenroll_by còn hiệu lực
    DB-->>Server: còn hạn
    Server-->>U: Đăng nhập thành công, kèm cảnh báo "Vui lòng enroll WebAuthn trước ngày X, SMS OTP sẽ ngừng được chấp nhận"

    Note over U,Server: Sau khi U enroll WebAuthn thành công trong hạn, MFA_ENROLLMENT (sms) bị vô hiệu, must_reenroll_by được xóa

    U->>Server: Đăng nhập bằng SMS OTP sau khi đã quá hạn 14 ngày
    Server->>DB: Kiểm tra must_reenroll_by đã quá hạn
    DB-->>Server: đã quá hạn
    Server-->>U: Từ chối SMS OTP, bắt buộc enroll WebAuthn ngay trước khi được cấp lại quyền truy cập
```
