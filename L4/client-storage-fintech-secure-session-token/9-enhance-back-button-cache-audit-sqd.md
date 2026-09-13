# Enhance sequence — Audit cache nhạy cảm và chặn xem lại bằng nút Back sau logout

Đây là **enhance**, cùng ý tưởng flow "logout" ở base nhưng bổ sung phần audit và chặn rò rỉ qua các lớp cache của trình duyệt. Sau khi logout, hệ thống phải đảm bảo không nơi nào (bfcache, service worker cache, HTTP cache) còn giữ lại dữ liệu nhạy cảm như số dư, số tài khoản, lịch sử giao dịch để có thể xem lại bằng nút Back. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant API as Balance/Transaction API
    participant Audit as CACHE_AUDIT_ENTRY
    participant SW as Service Worker cache (nếu có)
    participant Browser as bfcache của trình duyệt

    Note over Audit: Trước khi enhance, đội bảo mật rà soát và liệt kê từng bề mặt có nguy cơ giữ lại dữ liệu nhạy cảm
    Audit->>Audit: Ghi CACHE_AUDIT_ENTRY(surface=bfcache, sensitive_data_type=balance, mitigation=pagehide/unload clear)
    Audit->>Audit: Ghi CACHE_AUDIT_ENTRY(surface=service_worker_cache, sensitive_data_type=transaction_history, mitigation=cache.delete())
    Audit->>Audit: Ghi CACHE_AUDIT_ENTRY(surface=http_cache, sensitive_data_type=account_number, mitigation=Cache-Control no-store)

    User->>App: Xem số dư, lịch sử giao dịch
    App->>API: GET balance, transactions
    API-->>App: Trả dữ liệu kèm header Cache-Control no-store, để trình duyệt không lưu HTTP cache cho các response này

    User->>App: Bấm đăng xuất
    App->>App: Lắng nghe sự kiện pagehide/visibilitychange để chủ động xoá state nhạy cảm khỏi DOM và bộ nhớ trước khi trang có thể bị đưa vào bfcache
    App->>SW: Nếu có service worker cache riêng cho dữ liệu này, chủ động cache.delete() các entry liên quan tài khoản vừa logout

    User->>Browser: Bấm nút Back sau khi đã đăng xuất
    alt Trang được phục hồi từ bfcache
        Browser-->>App: Kích hoạt sự kiện pageshow với persisted=true
        App->>App: Phát hiện không còn access_token hợp lệ trong bộ nhớ, chủ động điều hướng lại về màn hình đăng nhập thay vì hiển thị dữ liệu cũ trong DOM
    else Trang tải lại bình thường
        App->>API: Gọi lại API, bị từ chối vì session đã revoke, điều hướng về đăng nhập
    end
```
