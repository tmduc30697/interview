# Enhance sequence — Tra cứu audit log immutable để điều tra sự cố

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 4 của đề bài: mọi thay đổi config (dù qua optimistic lock thông thường hay đường khẩn cấp) đều được ghi vào `CONFIG_AUDIT_LOG` không thể xóa/sửa, phục vụ điều tra khi 1 thay đổi config gây ảnh hưởng không mong muốn.

```mermaid
sequenceDiagram
    actor Investigator as Kỹ sư điều tra sự cố
    participant Dash as Config Dashboard
    participant Log as CONFIG_AUDIT_LOG (immutable, append-only)

    Investigator->>Dash: Yêu cầu xem toàn bộ lịch sử thay đổi của feature_flag_x
    Dash->>Log: SELECT * FROM CONFIG_AUDIT_LOG WHERE config_item_id = feature_flag_x ORDER BY changed_at
    Log-->>Dash: Danh sách đầy đủ, gồm cả bản ghi is_emergency_override=true lúc buộc lưu
    Dash-->>Investigator: Hiển thị timeline, ai sửa, khi nào, giá trị trước/sau, có phải action khẩn cấp không

    Investigator->>Dash: Thử xóa/sửa 1 bản ghi audit log bị nghi ngờ (kiểm tra tính immutable)
    Dash->>Log: Kiểm tra quyền, hệ thống không cung cấp API xóa/sửa cho bảng CONFIG_AUDIT_LOG
    Log-->>Dash: Từ chối, audit log là append-only
    Dash-->>Investigator: Không thể xóa/sửa lịch sử, đảm bảo dữ liệu điều tra luôn đáng tin cậy
```
