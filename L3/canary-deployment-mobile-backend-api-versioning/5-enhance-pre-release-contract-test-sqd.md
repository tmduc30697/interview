# Enhance sequence — Pre-release contract test (test đủ tập phiên bản tối thiểu)

Đây là **enhance**, cùng flow "Pre-release contract test" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 1 của đề bài: xác định tập phiên bản app tối thiểu cần hỗ trợ dựa trên số liệu user thực tế, và test hợp đồng API với từng phiên bản đó.

```mermaid
sequenceDiagram
    actor Backend as Backend Engineer
    participant Analytics as User Version Analytics
    participant CI as CI/CD
    participant Test as API_CONTRACT_TEST runner

    Backend->>CI: Chuẩn bị release backend version mới
    CI->>Analytics: Lấy phân bố phiên bản app đang được user thực tế dùng
    Analytics-->>CI: Danh sách MIN_SUPPORTED_APP_VERSION (vd v3.2, v4.0, v4.5 — vẫn còn active_user_count đáng kể)
    loop Với mỗi phiên bản trong tập tối thiểu
        CI->>Test: Chạy contract test backend mới với đúng phiên bản app này
        Test-->>CI: Ghi contract_test_result cho phiên bản đó
    end
    alt Tất cả phiên bản đều pass
        CI-->>Backend: "Contract test pass cho toàn bộ tập phiên bản tối thiểu, sẵn sàng canary"
    else Có phiên bản fail (vd app cũ không parse được field mới)
        CI-->>Backend: Chặn release, yêu cầu sửa hợp đồng API hoặc thêm shim tương thích trước khi canary
    end
```
