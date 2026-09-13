# Enhance sequence — Admin impersonation

Đây là **enhance**, flow hoàn toàn mới so với base — nhân viên support/admin nội bộ cần xem dữ liệu của 1 tenant cụ thể để hỗ trợ, thông qua 1 phiên impersonation có kiểm soát (có lý do, có thời hạn) thay vì dùng chung 1 quyền truy cập không giới hạn tenant. Mọi hành động trong phiên đều được ghi vào audit log, đúng yêu cầu đề bài.

```mermaid
sequenceDiagram
    actor Admin as Support Admin
    participant AdminPortal
    participant API as API Server
    participant DB as Shared Database (đã bật RLS)
    participant Audit as Audit Log Service

    Admin->>AdminPortal: Yêu cầu xem dữ liệu Tenant X, nhập lý do hỗ trợ
    AdminPortal->>API: POST /api/impersonation/start { target_tenant_id, reason }
    API->>API: Xác minh admin có role support, không tự cấp quyền vượt tenant
    API->>DB: INSERT impersonation_session (admin_user_id, target_tenant_id, reason, started_at)
    API->>Audit: Ghi log impersonation_started
    API-->>AdminPortal: Token impersonation, tenant_id = X, hết hạn sau N phút
    AdminPortal->>API: GET /api/projects (kèm token impersonation)
    API->>DB: SET app.current_tenant_id = X (lấy từ token impersonation, không phải tenant của admin)
    DB-->>API: Rows chỉ thuộc tenant X
    API->>Audit: Ghi log từng thao tác đọc/ghi trong lúc impersonate
    API-->>AdminPortal: Dữ liệu của tenant X
    AdminPortal-->>Admin: Hiển thị dữ liệu, kèm banner báo đang impersonate
    Admin->>AdminPortal: Kết thúc phiên impersonation
    AdminPortal->>API: POST /api/impersonation/end
    API->>DB: UPDATE impersonation_session SET ended_at = now()
    API->>Audit: Ghi log impersonation_ended
```
