# Base sequence — Login (không giới hạn số lần thử)

Đây là **base**, flow "Login" ở trạng thái hiện tại — kiểm tra username/password, không đếm số lần sai, không phân biệt nguồn gọi. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (dò mật khẩu 1 tài khoản nhiều lần, hoặc dò nhiều tài khoản từ 1 nguồn) đều khai thác đúng việc endpoint này cho thử vô hạn lần.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (hoặc user gõ nhầm)
    participant API as Login API
    participant DB as USER store

    loop Có thể lặp lại vô hạn lần, không bị chặn
        Attacker->>API: POST /login (username, password thử)
        API->>DB: Tìm USER theo username, so khớp password_hash
        alt Sai
            DB-->>API: Không khớp
            API-->>Attacker: "Invalid credentials"
        else Đúng
            DB-->>API: Khớp
            API->>API: Tạo SESSION mới
            API-->>Attacker: Login thành công
        end
    end

    Note over API,DB: Không có counter theo username hay IP, không có delay/captcha, không có log cảnh báo pattern bất thường
```
