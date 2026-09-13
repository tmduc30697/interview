# Enhance sequence — Emergency org-wide recovery (break-glass)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đây là flow trung tâm của đề bài, gộp 3 yêu cầu còn lại: (1) khôi phục cấp tổ chức thay vì xử lý từng user khi IdP hỏng hoàn toàn, (2) dùng `BREAK_GLASS_ACCOUNT` đã thiết lập sẵn từ trước kèm giám sát chặt (`AUDIT_LOG`), và (3) phải phân biệt sự cố tạm thời với việc thực sự cần khôi phục khẩn cấp trước khi kích hoạt quy trình rủi ro cao này.

```mermaid
sequenceDiagram
    actor Employees
    participant App as SaaS App
    participant Monitor as Outage Monitor
    participant Incident as OUTAGE_INCIDENT store
    actor BGAdmin as Org Admin (dùng break-glass)
    participant BG as BREAK_GLASS_ACCOUNT store
    participant Audit as AUDIT_LOG
    participant DB as SSO_CHANGE_REQUEST / IDP_CONFIG store

    Employees->>App: Thử đăng nhập qua IdP của org
    App-->>Monitor: Báo lỗi xác thực/timeout hàng loạt từ cùng 1 org
    Monitor->>Incident: Tạo/cập nhật OUTAGE_INCIDENT, đo duration
    alt Duration ngắn, tự phục hồi (transient)
        Incident->>Incident: classification=transient
        Monitor-->>BGAdmin: Không kích hoạt quy trình khẩn cấp
    else Duration vượt ngưỡng, không tự phục hồi
        Incident->>Incident: classification=confirmed_outage
        Monitor-->>BGAdmin: Cảnh báo, mở đường vào break-glass cho org
        BGAdmin->>App: Đăng nhập bằng BREAK_GLASS_ACCOUNT + MFA riêng
        App->>BG: Xác thực credential + MFA
        BG-->>App: Hợp lệ
        App->>Audit: Ghi log sử dụng break-glass (actor, thời điểm, hành động)
        BGAdmin->>DB: Tạo SSO_CHANGE_REQUEST (type=emergency_recovery, cite OUTAGE_INCIDENT)
        DB->>DB: Duyệt nhanh (expedited) nhưng vẫn ghi nhận đầy đủ
        DB->>DB: Kích hoạt IDP_CONFIG khôi phục (IdP mới hoặc tạm thời) cho toàn org
        DB-->>Employees: Toàn bộ nhân viên đăng nhập lại được (xem flow SSO Login)
        Note over Audit: Post-incident review dựa trên AUDIT_LOG sau khi khôi phục xong
    end
```
