# Enhance sequence — Test 2 booking chồng lấn lock theo thứ tự ngược nhau

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 5 của đề bài: dựng 2 transaction giả lập chồng lấn lock theo thứ tự ngày ngược nhau, cùng tranh chấp phòng cuối cùng còn trống, xác nhận hệ thống tự phát hiện/rollback một bên (do hết phòng khi đến lượt, không phải do deadlock — deadlock đã bị loại trừ nhờ chuẩn hóa thứ tự lock) và request còn lại thành công.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant BookA as Booking A (yêu cầu [10/9, 11/9, 12/9], cần 1 phòng cuối cùng)
    participant BookB as Booking B (yêu cầu [12/9, 11/9, 10/9], cũng cần 1 phòng cuối cùng)
    participant DB as Database (lock order đã chuẩn hóa)
    participant Log as DEADLOCK_TEST_LOG

    Test->>DB: Setup available_count=1 cho cả 3 ngày (10/9, 11/9, 12/9)
    Test->>BookA: Khởi chạy Booking A cùng lúc với Booking B
    Test->>BookB: Khởi chạy Booking B cùng lúc với Booking A

    par Chạy song song, cùng chạm 3 ngày trùng nhau
        BookA->>DB: BEGIN, lock theo thứ tự đã chuẩn hóa [10/9, 11/9, 12/9]
    and
        BookB->>DB: BEGIN, lock theo thứ tự đã chuẩn hóa [10/9, 11/9, 12/9] (giống Booking A dù request ngược)
    end

    Note over DB: Vì cùng thứ tự lock, Booking B chỉ phải chờ Booking A giải phóng lock, không có vòng chờ chéo -> không deadlock
    DB->>DB: Booking A vào trước, kiểm tra available_count=1 đủ, trừ còn 0, COMMIT
    DB->>DB: Booking B vào sau, kiểm tra available_count=0 không đủ, tự rollback theo logic nghiệp vụ (hết phòng)
    DB-->>Test: Booking A commit thành công, Booking B rollback với lỗi "hết phòng"

    Test->>Log: Ghi DEADLOCK_TEST_LOG (booking_id=B, blocking_booking_id=A, locked_resource=room_inventory 3 ngày, outcome=rolled_back)
    Test->>Test: assert Booking A thành công, Booking B bị rollback rõ ràng (không phải do deadlock), available_count cuối cùng = 0
```
