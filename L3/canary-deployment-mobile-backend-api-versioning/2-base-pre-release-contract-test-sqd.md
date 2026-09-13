# Base sequence — Pre-release contract test (chỉ test app mới nhất)

Đây là **base**, flow "Test hợp đồng API trước khi canary" ở trạng thái hiện tại — chỉ test với phiên bản app mới nhất. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 1 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    actor Backend as Backend Engineer
    participant CI as CI/CD
    participant Test as API_CONTRACT_TEST runner
    participant LatestApp as App version mới nhất

    Backend->>CI: Chuẩn bị release backend version mới
    CI->>Test: Chạy contract test với LatestApp
    Test->>LatestApp: Gọi API theo hợp đồng hiện tại
    LatestApp-->>Test: Parse response thành công
    Test-->>CI: result=pass
    CI-->>Backend: "Contract test pass, sẵn sàng canary"
    Note over Test,LatestApp: Không test với bất kỳ phiên bản app cũ nào đang được người dùng thực tế sử dụng — nếu backend đổi format response, app cũ có thể parse sai mà không ai biết trước khi canary
```
