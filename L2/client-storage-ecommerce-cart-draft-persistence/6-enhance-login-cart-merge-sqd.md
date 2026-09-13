# Enhance sequence — Merge giỏ hàng local với giỏ hàng server khi đăng nhập

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance. Khi user đăng nhập, giỏ hàng guest trong localStorage phải được merge đúng với giỏ hàng đã có sẵn trên server của tài khoản đó, theo quy tắc ưu tiên rõ ràng, không ghi đè ngầm định gây mất dữ liệu. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant LS as localStorage (LOCAL_CART_ITEM)
    participant API as Cart API
    participant DB as Server DB
    participant MergeLog as CART_MERGE_LOG

    User->>SPA: Đăng nhập thành công
    SPA->>LS: Đọc toàn bộ LOCAL_CART_ITEM hiện có (giỏ guest)
    SPA->>API: Lấy CART_ITEM hiện có trên server của tài khoản này
    API-->>SPA: Trả về giỏ hàng server

    loop Với mỗi sản phẩm xuất hiện ở cả 2 bên
        alt Số lượng khác nhau nhưng không mâu thuẫn nghiêm trọng
            SPA->>SPA: Áp quy tắc mặc định, ví dụ cộng dồn số lượng hoặc giữ số lượng lớn hơn
        else Chênh lệch lớn hoặc giá đã đổi giữa lúc thêm ở local và giá server hiện tại
            SPA-->>User: Hỏi rõ user chọn giữ giỏ local, giỏ server, hay cộng dồn
        end
        SPA->>MergeLog: Ghi CART_MERGE_LOG(local_quantity, server_quantity, resolution)
    end

    SPA->>API: Ghi kết quả merge cuối cùng lên CART_ITEM trên server
    API->>DB: Cập nhật giỏ hàng server theo kết quả merge
    API-->>SPA: Xác nhận giỏ hàng đã hợp nhất
    SPA->>LS: Xoá LOCAL_CART_ITEM vì đã merge xong lên server
```
