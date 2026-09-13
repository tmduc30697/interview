# Sequence - Enhance: Xác nhận thanh toán sát thời điểm hold hết hạn

Đây là flow **enhance hoàn toàn mới**, mô tả race giữa job tự động nhả ghế hết hạn và request xác nhận thanh toán đến ngay sát thời điểm hết hạn. Transaction xác nhận thanh toán phải kiểm tra hold vẫn còn hiệu lực (`held_by_user_id` khớp và chưa quá `held_until`) ngay trước khi chốt vé; nếu đã hết hạn phải từ chối rõ ràng và tự động hoàn tiền nếu cổng thanh toán bên ngoài đã báo thành công trước khi hệ thống kịp phát hiện hết hạn. Đáp ứng **yêu cầu 2** của đề bài.

```mermaid
sequenceDiagram
    participant U as Khách hàng
    participant API as Ticketing API
    participant Pay as Cổng thanh toán
    participant DB as Database
    participant Job as Job nhả ghế hết hạn

    Note over DB: SEAT A12: status=held, held_by_user_id=U, held_until=12:05:00.000

    U->>API: Xác nhận thanh toán lúc 12:04:59.950 (sát hạn)
    API->>Pay: Gửi yêu cầu thanh toán
    par Job nền chạy song song
        Job->>DB: Quét ghế có held_until < now(), gặp A12 lúc 12:05:00.010
        Job->>DB: UPDATE SEAT SET status='available' WHERE seat_id='A12' AND status='held' AND held_until < now()
        DB-->>Job: affected_rows=1, A12 đã bị nhả
    and
        Pay-->>API: Thanh toán thành công lúc 12:05:00.200
    end
    API->>DB: Transaction chốt vé, kiểm tra lại SEAT A12: status='held' AND held_by_user_id=U AND held_until > now() trước khi UPDATE status='sold'
    DB-->>API: Điều kiện không khớp (status đã là 'available' do job vừa nhả)
    Note over API: Phát hiện hold đã hết hạn ngay trước khi chốt, không chốt vé
    API->>Pay: Yêu cầu hoàn tiền cho giao dịch vừa thanh toán thành công
    Pay-->>API: Hoàn tiền thành công
    API-->>U: Ghế A12 đã hết hạn giữ chỗ, đã hoàn tiền, mời chọn ghế khác
```
