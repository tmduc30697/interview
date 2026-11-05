# Giữ chỗ tồn kho cho sản phẩm flash sale

**Hệ thống:** Sàn thương mại điện tử mở bán flash sale một sản phẩm số lượng giới hạn, hàng nghìn request mua hàng gửi tới cùng lúc trong vài giây đầu mở bán.

**Vai trò của flow:** Distributed lock theo từng sản phẩm (per-SKU) đảm bảo tại một thời điểm chỉ một tiến trình được đọc-và-trừ tồn kho cho sản phẩm đó, tránh oversell do nhiều request đọc cùng số tồn kho rồi cùng trừ.

**Yêu cầu cụ thể:**
- Lock phải được khóa ở granularity đúng mức (per-SKU, không lock toàn catalog gây nghẽn cổ chai, không lock quá lỏng theo warehouse chung gây oversell chéo sản phẩm), với TTL đủ ngắn để nhả nhanh cho request tiếp theo nhưng đủ dài để hoàn tất thao tác trừ kho — quá ngắn thì lock hết hạn giữa chừng gây 2 tiến trình cùng trừ, quá dài thì hàng nghìn request xếp hàng chờ gây timeout hàng loạt.
- Nếu tiến trình giữ lock crash ngay sau khi đã trừ số dư tồn kho trong DB nhưng chưa kịp tạo record đơn hàng/release lock, phải có cơ chế phát hiện trạng thái dở dang này khi lock hết hạn (ví dụ đối chiếu số đã trừ với đơn hàng tồn tại tương ứng) để hoàn lại tồn kho bị "kẹt" thay vì mất vĩnh viễn.
- Dùng fencing token gắn với lock để tránh trường hợp một tiến trình bị treo (GC pause, network delay) tưởng mình còn giữ lock, ghi trừ kho sau khi lock đã hết hạn và bị tiến trình khác giành mất — thao tác trừ kho với token cũ phải bị service tồn kho từ chối.
- Với hàng nghìn request cạnh tranh cùng 1 lock, phải có chiến lược fail-fast cho request không giành được lock trong thời gian ngắn (trả "hết hàng/thử lại" thay vì xếp hàng chờ vô hạn), tránh hàng đợi lock phình to làm nghẽn toàn hệ thống.
- Retry của client (do timeout network) không được gây trừ kho 2 lần cho cùng một yêu cầu mua — cần idempotency key gắn với request mua hàng độc lập với cơ chế lock, vì lock chỉ đảm bảo tuần tự xử lý chứ không đảm bảo request không bị gửi lặp.
