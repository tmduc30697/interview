# Leader coordinator cho cache invalidation tránh thundering herd

**Hệ thống:** Hệ thống cache nhiều lớp cho một trang tin tức lớn, khi cache miss đồng loạt (ví dụ sau khi publish bài mới) hàng trăm request cùng dội xuống DB.

**Vai trò của flow:** Dùng leader election/distributed lock để chỉ một request/instance được phép đi warm-up cache/query DB, các request khác chờ hoặc dùng dữ liệu cũ.

**Yêu cầu cụ thể:**
- Chỉ request đầu tiên "giành" được lock cho một cache key mới được query DB và ghi lại cache; các request đến sau trong lúc lock còn giữ phải chờ ngắn (vài trăm ms) hoặc trả stale data nếu có, không tự query DB song song.
- Lock phải tự hết hạn nếu request giữ lock bị treo/crash, để tránh các request khác chờ vô hạn.
- Có cơ chế "single flight" nội bộ trong từng instance kết hợp với lock phân tán giữa các instance để giảm số round-trip tới coordinator.
- Đo lường: số lượng DB query giảm được so với baseline không có coordinator (mục tiêu giảm >90% query trùng lặp trong 1 giây đầu sau cache miss).
- Xử lý network partition giữa instance và Redis: instance phải có fallback (ví dụ tự query DB có giới hạn rate) khi không kết nối được coordinator, tránh outage toàn phần.
