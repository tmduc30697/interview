# Enhance sequence — Checkpoint cho job dài, tránh gửi trùng khi xử lý lại

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 3 và yêu cầu 4** của đề bài: job gửi email hàng loạt có thể để lại tác động một phần khi bị ngắt giữa chừng, nên phải lưu checkpoint (đã gửi tới người thứ mấy) để lần xử lý lại sau không gửi trùng. Vì job này dài hơn grace period tối đa cho phép, worker chủ động lưu checkpoint giữa chừng thay vì cố chờ chạy xong trong grace period.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W1 as Worker A
    participant Store as Checkpoint Store
    participant W2 as Worker B (xử lý lại sau)

    Queue->>W1: Deliver message (job=bulk_email_88, danh sách 10000 người nhận)
    W1->>W1: Bắt đầu gửi email lần lượt

    loop Mỗi 500 email đã gửi thành công
        W1->>Store: UPDATE checkpoint = {sent_up_to_index: 500}
    end

    Note over W1: Job này ước tính mất 12 phút, dài hơn grace_period_seconds tối đa (30 giây) nên không cố chờ xong trong grace period
    W1->>W1: Nhận tín hiệu shutdown khi đã gửi tới người thứ 3200
    W1->>Store: UPDATE checkpoint = {sent_up_to_index: 3200} (lưu ngay trước khi hết grace period)
    W1->>Queue: Không ACK (để message quay lại queue cho worker khác tiếp tục)
    W1->>W1: Hết grace period, tắt hẳn

    Queue->>W2: Deliver lại message (job=bulk_email_88)
    W2->>Store: SELECT checkpoint WHERE job_id=bulk_email_88
    Store-->>W2: {sent_up_to_index: 3200}
    W2->>W2: Tiếp tục gửi từ người thứ 3201 trở đi, không gửi trùng cho 3200 người đã nhận
    W2->>Store: Ghi checkpoint cuối cùng khi gửi xong toàn bộ 10000 người
    W2->>Queue: ACK job=bulk_email_88
```
