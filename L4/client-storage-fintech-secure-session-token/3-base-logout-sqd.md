# Base sequence — Logout (chỉ xoá tab hiện tại, không có audit cache)

Đây là **base**, flow đăng xuất ở trạng thái trước enhance. Logout chỉ xoá localStorage của đúng tab đang bấm, các tab khác đang mở với cùng tài khoản vẫn còn access_token hợp lệ trong localStorage của chúng (localStorage dùng chung theo origin nhưng flow logout ở base không chủ động thông báo cho tab khác). Ngoài ra không có xử lý gì cho cache-control hay bfcache, nên bấm Back sau khi logout có thể thấy lại trang có dữ liệu nhạy cảm từ bộ nhớ cache trình duyệt.

```mermaid
sequenceDiagram
    actor User as Người dùng (Tab A)
    participant TabA as Tab A
    participant LS as localStorage (dùng chung origin)
    actor OtherUser as Người dùng (Tab B, cùng tài khoản)
    participant TabB as Tab B

    User->>TabA: Xem số dư, lịch sử giao dịch (dùng access_token từ localStorage)
    User->>TabA: Bấm đăng xuất
    TabA->>LS: Xoá access_token, refresh_token khỏi localStorage
    TabA-->>User: Chuyển về màn hình đăng nhập

    Note over TabB: Tab B đang mở cùng tài khoản không nhận được thông báo gì
    OtherUser->>TabB: Vẫn thao tác bình thường ở Tab B với access_token cũ còn hiệu lực

    User->>TabA: Bấm nút Back trên trình duyệt
    TabA-->>User: Có thể hiển thị lại trang số dư/giao dịch từ cache trình duyệt (bfcache), dù đã đăng xuất
```
