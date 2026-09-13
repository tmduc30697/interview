# Enhance sequence — Kiểm tra cô lập giữa 2 tab dùng 2 tài khoản khác nhau

Đây là **enhance**, flow hoàn toàn mới, mô tả kịch bản kiểm thử khi nhân viên hỗ trợ vô tình mở 2 tab với 2 tài khoản khác nhau để test. Vì sessionStorage và biến bộ nhớ JS cô lập theo từng tab nên đây là hành vi đúng, nhưng phải kiểm tra kỹ để đảm bảo không có state nào rò rỉ qua cơ chế dùng chung khác, ví dụ kênh BroadcastChannel của một tính năng không liên quan tới auth. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor UserX as Nhân viên hỗ trợ (Tab A, tài khoản X)
    participant TabA as Tab A (ACCESS_TOKEN_MEMORY của X, SESSION_UI_STATE của X)
    actor UserY as Nhân viên hỗ trợ (Tab B, tài khoản Y)
    participant TabB as Tab B (ACCESS_TOKEN_MEMORY của Y, SESSION_UI_STATE của Y)
    participant AuthChannel as AUTH_BROADCAST_EVENT (kênh auth, có kiểm tra origin_tab_id/user_id)
    participant OtherChannel as BroadcastChannel của tính năng khác, ví dụ thông báo real-time

    UserX->>TabA: Đăng nhập tài khoản X
    UserY->>TabB: Đăng nhập tài khoản Y ở tab khác

    Note over TabA,TabB: ACCESS_TOKEN_MEMORY và SESSION_UI_STATE cô lập đúng theo từng tab, đây là hành vi mong muốn

    UserX->>TabA: Đăng xuất tài khoản X
    TabA->>AuthChannel: Phát sự kiện logout kèm user_id=X
    AuthChannel-->>TabB: Nhận được sự kiện, nhưng kiểm tra user_id trong event không khớp session hiện tại của Tab B (Y)
    TabB->>TabB: Bỏ qua sự kiện logout của tài khoản X, không tự đăng xuất tài khoản Y

    rect rgb(255, 235, 235)
    Note over OtherChannel: Trường hợp rủi ro cần test kỹ, nếu 1 tính năng khác dùng chung BroadcastChannel không đặt tên riêng và không kiểm tra user_id
    UserX->>TabA: Thao tác khác (ví dụ chọn 1 filter dashboard) vô tình phát qua OtherChannel
    OtherChannel-->>TabB: Nếu kênh này không kiểm tra user_id, Tab B (tài khoản Y) có thể vô tình nhận nhầm state của tài khoản X
    end

    Note over TabA,TabB: Biện pháp, đặt tên kênh BroadcastChannel auth riêng biệt và luôn kèm/kiểm tra user_id trong mọi message trước khi áp dụng ở tab nhận
```
