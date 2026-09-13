# Enhance sequence — Login (phân biệt thiết bị mới thật sự với thiết bị cũ cài lại app)

Đây là **enhance** của flow `login` đã có ở base. So với base (mọi thiết bị đăng nhập thành công đều có toàn quyền như nhau), nay thiết bị mới được gán `trust_level=basic_view` mặc định, còn nếu hệ thống nhận diện được `device_binding_key` khớp với thiết bị đã từng đạt trust cao trước đó (cài lại app, chưa xoá secure storage), trust được khôi phục sau một bước xác minh nhẹ thay vì reset về mức thấp nhất. Đáp ứng yêu cầu 1 và 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant Auth as Auth Service
    participant DB as Database
    participant Log as TRUST_LEVEL_CHANGE_LOG

    User->>App: Nhập username/password
    App->>Auth: Xác thực password + MFA
    Auth-->>App: Xác thực thành công

    App->>App: Đọc device_binding_key từ secure storage của hệ điều hành (nếu còn tồn tại)
    App->>DB: Gửi kèm device_fingerprint + device_binding_key (nếu có)

    alt Không có device_binding_key hoặc không khớp bản ghi nào (thiết bị mới thật sự)
        DB->>DB: INSERT DEVICE mới (trust_level=basic_view, maturity_window_ends_at=now+7 ngày)
        DB->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(reason=maturity_window_passed sẽ áp dụng sau, hiện tại new_trust_level=basic_view)
        Auth-->>App: Trả access token, chỉ được xem thông tin cơ bản
    else device_binding_key khớp thiết bị cũ đã từng đạt high_value_transfer (cài lại app)
        DB->>App: Yêu cầu xác minh nhẹ bổ sung (vd MFA lần 2 hoặc sinh trắc học có sẵn trên máy)
        App-->>DB: Xác minh nhẹ thành công
        DB->>DB: Khôi phục DEVICE.trust_level = mức trust trước đó, status=active
        DB->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(reason=reinstall_recognized, new_trust_level=mức đã khôi phục)
        Auth-->>App: Trả access token với trust đã khôi phục, không phải chờ lại maturity window
    end

    App-->>User: Đăng nhập thành công
```
