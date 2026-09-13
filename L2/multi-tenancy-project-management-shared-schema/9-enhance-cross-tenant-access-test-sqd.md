# Enhance sequence — Cross-tenant access test

Đây là **enhance**, flow hoàn toàn mới so với base — test tự động cố tình dùng token của tenant A để truy cập tài nguyên của tenant B qua các endpoint quan trọng, đảm bảo luôn bị chặn 403/404. Đây chính là yêu cầu "viết test tự động, không chỉ manual test" trong đề bài, chạy trong CI trên mọi lần deploy chứ không chỉ dựa vào việc UI không hiển thị link.

```mermaid
sequenceDiagram
    participant TestSuite as Automated Test Suite
    participant API as API Server
    participant DB as Shared Database (đã bật RLS)

    TestSuite->>TestSuite: Đăng nhập User A (tenant A), lấy token A
    TestSuite->>TestSuite: Chuẩn bị resource id thuộc tenant B (project, file, report...)
    TestSuite->>API: GET /api/projects/{project_id_cua_tenant_B} kèm token A
    API->>DB: SELECT * FROM projects WHERE id = :id (tenant context = A)
    Note over DB: RLS lọc theo tenant A, row thuộc tenant B trở thành vô hình
    DB-->>API: Không tìm thấy row nào
    API-->>TestSuite: 404 Not Found
    TestSuite->>TestSuite: Assert response phải là 403/404, không được là 200
    Note over TestSuite: Lặp lại test này cho mọi endpoint nhạy cảm, gồm cả tải file và xem report
    Note over TestSuite: Test chạy trong CI ở mọi lần deploy, không chỉ manual QA
```
