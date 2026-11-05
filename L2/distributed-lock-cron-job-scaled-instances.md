# Cron/scheduled job chạy trên nhiều instance service (horizontally scaled)

**Hệ thống:** Một service backend được scale ra nhiều instance (pod) để chịu tải, nhưng có vài job định kỳ (gửi báo cáo, dọn dữ liệu) chỉ được chạy đúng 1 lần mỗi chu kỳ.

**Vai trò của flow:** Dùng distributed lock (lease-based, TTL) trên Redis/ZooKeeper để đảm bảo chỉ một instance giành được lock mới được chạy job, các instance khác skip.

**Yêu cầu cụ thể:**
- Lock phải có TTL (lease) rõ ràng; instance giữ lock phải renew (heartbeat) trước khi TTL hết, nếu không renew kịp thì lock tự nhả cho instance khác.
- Xử lý crash: nếu instance giữ lock crash giữa job (không kịp release lock), lock phải tự hết hạn sau TTL và job có thể chạy lại bởi instance khác — không bị "khoá chết" vĩnh viễn.
- Chống trường hợp instance cũ tưởng mình còn giữ lock (do GC pause/network delay) tiếp tục ghi dữ liệu sau khi lock đã hết hạn và bị instance khác giành mất (fencing token / lock version check trước khi ghi kết quả job).
- Log rõ instance nào giữ lock, thời điểm acquire/release/renew, để debug khi nghi ngờ job chạy trùng.
- Đo SLA: thời gian tối đa từ lúc lock cũ hết hạn tới lúc instance mới giành được lock mới (failover time) phải dưới một ngưỡng cụ thể (ví dụ 10s).
