# Enhance sequence — Đường khẩn cấp "buộc lưu", bỏ qua version check

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 3 của đề bài: khi vẫn thực sự xảy ra conflict đúng ngay trên item cần tắt khẩn cấp (không chỉ là xung đột giả), kỹ sư có thể bấm "buộc lưu" để bỏ qua version check, ưu tiên tốc độ xử lý sự cố hơn rủi ro ghi đè nhầm — nhưng action này bắt buộc phải được log rõ ràng và cảnh báo.

```mermaid
sequenceDiagram
    actor EngineerA as Kỹ sư A (đang xử lý sự cố nghiêm trọng)
    actor EngineerC as Kỹ sư C
    participant Dash as Config Dashboard
    participant FlagDB as CONFIG_ITEM "feature_flag_x"
    participant Log as CONFIG_AUDIT_LOG

    EngineerA->>Dash: Mở feature_flag_x, nhận value=on, version=9
    EngineerC->>Dash: Cũng vừa sửa feature_flag_x (vd đổi rollout %), lưu thành công, version=10

    EngineerA->>Dash: Tắt flag khẩn cấp, gửi version=9 (đã lỗi thời)
    Dash->>FlagDB: UPDATE CONFIG_ITEM SET value=off, version=10 WHERE id=feature_flag_x AND version=9
    FlagDB-->>Dash: 0 row affected (version hiện tại đã là 10, đây là conflict thật)
    Dash-->>EngineerA: Từ chối, hiển thị cảnh báo "config đã đổi, bạn có muốn BUỘC LƯU không (bỏ qua kiểm tra xung đột)?"

    EngineerA->>Dash: Xác nhận "Buộc lưu" (chấp nhận rủi ro vì đang là sự cố khẩn cấp)
    Dash->>FlagDB: UPDATE CONFIG_ITEM SET value=off, version=11 WHERE id=feature_flag_x (không kèm điều kiện version)
    FlagDB-->>Dash: 1 row affected, ghi đè thành công bất kể version hiện tại
    Dash->>Log: Ghi CONFIG_AUDIT_LOG (changed_by=A, old_value=<giá trị bị ghi đè>, new_value=off, is_emergency_override=true, note="bỏ qua version check do sự cố khẩn cấp")
    Dash-->>EngineerA: Flag đã tắt, kèm cảnh báo hiển thị lại rằng action này đã bỏ qua kiểm tra xung đột
    Note over Dash,Log: Vì action được đánh dấu is_emergency_override=true trong audit log, đội điều tra sau này luôn phân biệt được đâu là lưu bình thường, đâu là buộc lưu khẩn cấp
```
