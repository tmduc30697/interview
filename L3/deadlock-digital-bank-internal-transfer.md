# Chuyển tiền nội bộ giữa các tài khoản ngân hàng số

**Hệ thống:** Một app ngân hàng số cho phép user chuyển tiền giữa các tài khoản của chính họ và tài khoản người khác trong cùng hệ thống.

**Vai trò của flow:** Transaction cập nhật số dư của 2 tài khoản (trừ bên gửi, cộng bên nhận) phải lock đúng thứ tự để tránh deadlock khi nhiều lệnh chuyển tiền ngược hướng nhau chạy song song.

**Yêu cầu cụ thể:**
- Mô tả cụ thể kịch bản: transaction A chuyển từ account 1 → account 2, transaction B chuyển từ account 2 → account 1 chạy đồng thời; nếu mỗi transaction lock theo thứ tự "tài khoản nguồn trước, tài khoản đích sau" thì sẽ deadlock — yêu cầu code phải lock theo thứ tự cố định (ví dụ theo account_id tăng dần) để loại trừ deadlock này.
- Chọn isolation level `READ COMMITTED` hoặc cao hơn cho transaction cập nhật số dư, giải thích vì sao `REPEATABLE READ` mặc định của một số DB (MySQL/InnoDB) có thể gây phạm vi lock rộng hơn cần thiết và tăng khả năng deadlock trên các dòng không liên quan.
- Khi DB trả lỗi deadlock (ví dụ mã lỗi 40P01 ở Postgres, 1213 ở MySQL), transaction bị rollback phải được retry tự động tối đa 3 lần với backoff, không được để lỗi văng lên user như một giao dịch thất bại vĩnh viễn.
- Đảm bảo tổng số dư hai tài khoản trước và sau giao dịch không đổi (invariant) dù có retry hay không — viết test giả lập 50 giao dịch ngược hướng chạy song song và assert invariant này.
- Ghi log đầy đủ thông tin deadlock (transaction nào, lock gì, thời điểm) để phục vụ debug, không chỉ log "transaction failed".
