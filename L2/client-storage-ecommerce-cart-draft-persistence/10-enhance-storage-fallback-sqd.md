# Enhance sequence — Fallback khi localStorage đầy hoặc bị chặn

Đây là **enhance**, flow hoàn toàn mới, xử lý trường hợp localStorage bị đầy (quota exceeded) hoặc bị user tắt (chế độ duyệt riêng tư/chặn storage). Hệ thống phải phát hiện tình trạng này, chuyển sang fallback in-memory only, và cảnh báo user mà không làm crash trang. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant LS as localStorage
    participant Mem as In-memory fallback cart
    participant Fallback as STORAGE_FALLBACK_STATE

    User->>SPA: Bấm thêm sản phẩm vào giỏ (guest)
    SPA->>LS: Thử ghi LOCAL_CART_ITEM

    alt Ghi thành công
        LS-->>SPA: OK
        SPA->>Fallback: Ghi local_storage_available=true, fallback_in_memory_active=false
    else Lỗi quota exceeded hoặc storage bị chặn (chế độ riêng tư)
        LS-->>SPA: Ném lỗi khi ghi (QuotaExceededError hoặc SecurityError)
        SPA->>SPA: Bắt lỗi, không để lỗi này làm crash trang
        SPA->>Mem: Chuyển sang giữ giỏ hàng tạm trong bộ nhớ (in-memory only) cho phiên hiện tại
        SPA->>Fallback: Ghi local_storage_available=false, fallback_in_memory_active=true
        SPA-->>User: Hiển thị cảnh báo, trình duyệt đang chặn lưu tạm nên giỏ hàng có thể mất khi đóng tab
    end

    Note over Mem: Trong chế độ fallback, giỏ hàng vẫn hoạt động bình thường trong phiên hiện tại nhưng không còn bền qua reload
```
