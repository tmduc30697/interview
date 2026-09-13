# Enhance sequence — Thông báo real-time khi có người vừa lưu thay đổi

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 5 của đề bài: các kỹ sư khác đang mở màn hình chỉnh sửa cùng 1 config được thông báo real-time (qua WebSocket) ngay khi có ai đó vừa lưu thay đổi, giảm khả năng họ submit dựa trên version đã lỗi thời.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A
    actor EngineerC as Kỹ sư C (đang mở cùng màn hình sửa)
    participant Dash as Config Dashboard
    participant WS as WebSocket Gateway
    participant SessionStore as CONFIG_EDIT_SESSION
    participant FlagDB as CONFIG_ITEM

    EngineerC->>Dash: Mở màn hình sửa feature_flag_x, nhận version=7
    Dash->>SessionStore: Ghi CONFIG_EDIT_SESSION (engineer=C, config_item=feature_flag_x, last_seen_version=7)
    Dash->>WS: Subscribe kênh cập nhật của feature_flag_x cho Engineer C

    EngineerA->>Dash: Lưu thành công feature_flag_x, version mới = 8
    Dash->>SessionStore: Tra các CONFIG_EDIT_SESSION đang mở feature_flag_x có last_seen_version cũ hơn 8
    SessionStore-->>Dash: Engineer C đang mở với last_seen_version=7 (đã lỗi thời)
    Dash->>WS: Publish sự kiện "feature_flag_x vừa được cập nhật lên version 8 bởi A"
    WS-->>EngineerC: Đẩy thông báo real-time lên màn hình đang mở

    EngineerC->>Dash: Thấy cảnh báo, chủ động tải lại config mới nhất trước khi sửa tiếp
    Dash->>FlagDB: SELECT value, version FROM CONFIG_ITEM WHERE id=feature_flag_x
    FlagDB-->>Dash: value mới nhất, version=8
    Dash-->>EngineerC: Cập nhật UI với version 8, Engineer C không còn submit dựa trên version 7 đã cũ
```
