# Enhance sequence — Đồng bộ giỏ hàng giữa nhiều tab

Đây là **enhance**, flow hoàn toàn mới, xử lý trường hợp mở nhiều tab cùng giỏ hàng guest. Khi 1 tab thêm/xoá item, các tab khác phải cập nhật badge số lượng qua `storage` event hoặc BroadcastChannel, đồng thời tránh tình trạng 1 tab ghi đè toàn bộ localStorage của tab khác làm mất thay đổi vừa thực hiện ở tab kia. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor UserA as Người dùng (Tab A)
    participant TabA as SPA Tab A
    participant LS as localStorage (LOCAL_CART_ITEM, dùng chung origin)
    participant Channel as storage event / BroadcastChannel
    participant TabB as SPA Tab B

    UserA->>TabA: Thêm sản phẩm X vào giỏ ở Tab A
    TabA->>LS: Đọc LOCAL_CART_ITEM hiện tại, chỉ cập nhật đúng item của sản phẩm X, không ghi đè toàn bộ danh sách
    TabA->>LS: Ghi lại LOCAL_CART_ITEM đã cập nhật
    TabA->>Channel: Phát CART_SYNC_EVENT(action=add, product_id=X, quantity, origin_tab_id=A)

    Channel-->>TabB: Nhận CART_SYNC_EVENT từ Tab A
    TabB->>LS: Đọc lại LOCAL_CART_ITEM mới nhất từ localStorage, không dùng bản cache cũ trong bộ nhớ Tab B
    TabB-->>UserA: Cập nhật badge số lượng giỏ hàng ở Tab B khớp với thay đổi từ Tab A

    Note over TabA,TabB: Vì mỗi thao tác chỉ cập nhật đúng item liên quan rồi ghi lại toàn bộ state mới nhất đọc từ localStorage, thay đổi gần nhất ở tab này không bị tab kia ghi đè mất
```
