# Enhance sequence — Phát hiện pattern tấn công diện rộng, alert real-time

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không log hay phân tích pattern của các lần login fail). Đáp ứng yêu cầu 5 của đề bài: log và alert khi phát hiện số lượng lớn login fail trong thời gian ngắn từ nhiều IP nhắm vào nhiều username, để team security theo dõi real-time.

```mermaid
sequenceDiagram
    participant API as Login API
    participant Attempt as LOGIN_ATTEMPT
    participant Detector as Attack Pattern Detector (job real-time)
    participant Alert as SECURITY_ALERT
    actor SecTeam as Team security

    loop Mỗi lần login fail
        API->>Attempt: Ghi LOGIN_ATTEMPT(success=false, ip, username, device_fingerprint)
    end

    loop Quét liên tục theo cửa sổ thời gian ngắn (ví dụ mỗi phút)
        Detector->>Attempt: Đếm distinct_ip_count và distinct_username_count trong window fail gần nhất

        alt distinct_ip_count và distinct_username_count đều vượt ngưỡng bất thường
            Detector->>Alert: Tạo SECURITY_ALERT(pattern_type=distributed_credential_stuffing, status=open)
            Alert-->>SecTeam: Cảnh báo real-time kèm số liệu IP/username liên quan
        else Chỉ 1 username bị nhắm với fail_count rất cao từ ít IP
            Detector->>Alert: Tạo SECURITY_ALERT(pattern_type=targeted_brute_force, status=open)
            Alert-->>SecTeam: Cảnh báo tài khoản cụ thể đang bị dò
        else Không có pattern bất thường
            Detector->>Detector: Không tạo alert
        end
    end

    SecTeam->>Alert: Xử lý xong, đánh dấu status=resolved
```
