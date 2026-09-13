# Enhance sequence — Xung đột giả không còn xảy ra nhờ version theo từng item

Đây là **enhance** của flow `cross-flag-false-conflict-emergency` đã có ở base. So với base (A bị chặn tắt flag X chỉ vì B lưu 1 config không liên quan), giờ mỗi `CONFIG_ITEM` có version riêng nên B lưu config khác không hề ảnh hưởng tới version của flag X — A tắt được flag ngay qua optimistic lock bình thường, không cần tới đường khẩn cấp. Đáp ứng trực tiếp yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A (đang xử lý sự cố)
    actor EngineerB as Kỹ sư B (sửa config không liên quan)
    participant Dash as Config Dashboard
    participant FlagDB as CONFIG_ITEM "feature_flag_x"
    participant OtherDB as CONFIG_ITEM "other_config"

    EngineerA->>Dash: Mở feature_flag_x, nhận value=on, version=7
    EngineerB->>Dash: Mở other_config (không liên quan), nhận value=A, version=20

    EngineerB->>Dash: Lưu other_config, gửi version=20
    Dash->>OtherDB: UPDATE CONFIG_ITEM SET value=..., version=21 WHERE id=other_config AND version=20
    OtherDB-->>Dash: 1 row affected, thành công
    Dash-->>EngineerB: Lưu thành công

    EngineerA->>Dash: Tắt feature_flag_x khẩn cấp, gửi version=7
    Dash->>FlagDB: UPDATE CONFIG_ITEM SET value=off, version=8 WHERE id=feature_flag_x AND version=7
    FlagDB-->>Dash: 1 row affected, thành công (version của feature_flag_x hoàn toàn độc lập với other_config)
    Dash-->>EngineerA: Tắt flag thành công ngay lập tức qua optimistic lock bình thường
    Note over FlagDB,OtherDB: Vì mỗi config item có version riêng, thay đổi không liên quan của B không còn chặn được thao tác khẩn cấp của A
```
