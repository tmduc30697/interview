# Cache nhiều tầng cho CMS/blog

**Hệ thống:** Hệ thống CMS/blog phục vụ bài viết qua nhiều tầng cache chồng lên nhau: CDN edge cache (gần user), cache tầng ứng dụng (application/object cache), và cache kết quả query DB (query cache) — mỗi tầng có TTL và cơ chế invalidate riêng.

**Vai trò của flow:** Đảm bảo khi 1 bài viết được sửa/xóa/publish, việc invalidate phải lan đúng và đủ qua toàn bộ các tầng cache, không chỉ dừng ở tầng gần nhất (application cache) mà bỏ sót CDN edge hoặc query cache khiến người đọc vẫn thấy nội dung cũ dù tầng app đã đúng.

**Yêu cầu cụ thể:**
- Khi biên tập viên sửa nội dung bài viết và publish lại, invalidation phải được gửi tới cả 3 tầng theo đúng thứ tự phụ thuộc (query cache trước, application cache sau, rồi purge CDN edge) — nếu purge CDN trước mà tầng dưới chưa cập nhật, request tiếp theo tới edge sẽ pull lại đúng nội dung cũ từ application cache và cache lại nó ở edge, coi như invalidate CDN vô nghĩa.
- Purge CDN edge cache thường không đồng bộ ngay lập tức trên toàn bộ điểm PoP (point of presence) toàn cầu — phải chấp nhận và công bố rõ "khoảng lan truyền" (propagation window) thực tế, đồng thời có cơ chế xác nhận purge đã tới đủ số PoP quan trọng trước khi coi invalidation hoàn tất.
- Với bài viết bị xóa hẳn (không chỉ sửa), phải đảm bảo tầng query cache (thường cache theo query như "danh sách bài mới nhất") cũng được invalidate, không chỉ cache theo ID bài viết đơn lẻ — nếu bỏ sót, bài đã xóa vẫn xuất hiện trong danh sách trang chủ dù trang chi tiết đã trả 404 đúng.
- Xử lý trường hợp một tầng cache invalidate thất bại (ví dụ API purge CDN timeout hoặc trả lỗi) — phải có retry với theo dõi trạng thái, và log rõ tầng nào invalidate thành công/thất bại để vận hành biết còn tầng nào đang phục vụ nội dung cũ, tránh tình trạng "tưởng đã invalidate hết" nhưng thực ra một tầng vẫn stale.
- Đo lường: với mỗi lần publish/sửa bài, track được "end-to-end staleness" — thời gian từ lúc publish tới lúc cả 3 tầng đều phản ánh đúng nội dung mới, phân theo từng tầng để biết tầng nào đang là nút thắt cổ chai của độ trễ invalidate.
