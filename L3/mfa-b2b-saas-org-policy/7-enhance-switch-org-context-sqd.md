# Enhance sequence — Switch org context (MFA áp theo đúng ngữ cảnh org đang active)

Đây là **enhance**, flow mới phát sinh từ đề bài — user có 1 email tham gia nhiều tổ chức với chính sách MFA khác nhau, chuyển ngữ cảnh làm việc giữa các tổ chức trong cùng phiên đăng nhập. Mỗi lần chuyển, hệ thống đánh giá lại MFA theo đúng `ORG_MFA_POLICY` của org đang được chọn, để user không thể né MFA chặt của 1 tổ chức bằng cách chuyển qua tổ chức khác lỏng hơn rồi vẫn thao tác trên dữ liệu nhạy cảm. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (thành viên của Org1 và Org2)
    participant Server
    participant DB as Database

    Note over U: Org1 bắt buộc MFA cho mọi role, Org2 không bắt buộc MFA

    U->>Server: Đăng nhập, chọn active_org_context=Org2 trước
    Server->>DB: SELECT ORG_MFA_POLICY WHERE org_id=Org2
    DB-->>Server: required=false
    Server->>DB: UPDATE SESSION SET active_org_context=Org2
    DB-->>Server: OK
    Server-->>U: Vào được Org2 chỉ với password, không cần MFA

    U->>Server: Switch context sang Org1 (vẫn cùng session đang đăng nhập)
    Server->>DB: SELECT ORG_MFA_POLICY WHERE org_id=Org1
    DB-->>Server: required=true
    Server->>DB: SELECT MFA_ENROLLMENT WHERE user_id=U
    DB-->>Server: chưa có, hoặc chưa xác thực MFA trong phiên này cho Org1

    Server-->>U: Chặn switch, yêu cầu xác thực MFA trước khi vào ngữ cảnh Org1
    U->>Server: Xác thực MFA thành công
    Server->>DB: UPDATE SESSION SET active_org_context=Org1, mfa_verified_for_context=true
    DB-->>Server: OK
    Server-->>U: Vào được Org1

    Note over Server,DB: Mỗi lần active_org_context đổi, server luôn tra lại policy của org đích, không tái sử dụng trạng thái MFA đã xác thực cho 1 org lỏng hơn
```
