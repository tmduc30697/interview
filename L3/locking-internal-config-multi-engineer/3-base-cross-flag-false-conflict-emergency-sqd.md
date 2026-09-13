# Base sequence — Xung đột giả chặn cả thao tác khẩn cấp

Đây là **base**, mô tả đúng kịch bản nêu ở yêu cầu 2 của đề bài: kỹ sư A đang tắt feature flag X do phát hiện lỗi khẩn cấp, đúng lúc kỹ sư B đang sửa 1 config khác không liên quan trong cùng nhóm nhưng version chung bị tăng do B lưu trước. Vì optimistic lock ở cấp cả nhóm và base chưa có đường xử lý khẩn cấp, A bị chặn hoàn toàn — đây chính là vấn đề nghiêm trọng mà việc tách version theo từng item và thêm đường khẩn cấp (enhance) phải giải quyết.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A (đang xử lý sự cố)
    actor EngineerB as Kỹ sư B (sửa config không liên quan)
    participant Dash as Config Dashboard
    participant DB as CONFIG_GROUP table

    EngineerA->>Dash: Mở nhóm config, nhận version = 12 (đang định tắt flag X)
    EngineerB->>Dash: Mở cùng nhóm config, cũng nhận version = 12

    EngineerB->>Dash: Lưu thay đổi config khác (không liên quan flag X), gửi version = 12
    Dash->>DB: UPDATE CONFIG_GROUP SET ..., version = 13 WHERE id = G AND version = 12
    DB-->>Dash: 1 row affected, thành công
    Dash-->>EngineerB: Lưu thành công, version mới = 13

    EngineerA->>Dash: Tắt flag X khẩn cấp, gửi version = 12 (đã lỗi thời do B vừa lưu)
    Dash->>DB: UPDATE CONFIG_GROUP SET flag_x = off, version = 13 WHERE id = G AND version = 12
    DB-->>Dash: 0 row affected (version hiện tại đã là 13)
    Dash-->>EngineerA: Từ chối lưu, báo "config đã được người khác cập nhật"
    Note over EngineerA,Dash: A bị chặn tắt flag khẩn cấp chỉ vì B lưu 1 config hoàn toàn không liên quan, và base chưa có đường xử lý khẩn cấp nào để bỏ qua tình huống này
```
