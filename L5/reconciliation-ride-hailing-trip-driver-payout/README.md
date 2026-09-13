# Ứng dụng gọi xe đối soát cuốc xe, tiền tài xế và hoa hồng

**Hệ thống:** Một ứng dụng gọi xe, khách đặt xe, tài xế hoàn tất cuốc, nền tảng thu tiền và chia lại cho tài xế sau khi trừ hoa hồng, một số cuốc bị hủy giữa chừng hoặc phát sinh tranh chấp giá.

**Vai trò của flow:** Đối soát giữa số tiền khách trả, số tiền tài xế được nhận, và hoa hồng nền tảng giữ lại, đảm bảo ba khoản này khớp đúng cho từng cuốc xe và không có cuốc nào bị tính sai/bỏ sót.

**Yêu cầu cụ thể:**
- Cuốc xe bị hủy giữa chừng sau khi tài xế đã di chuyển một đoạn có thể phát sinh phí hủy một phần cho tài xế nhưng không phải giá cuốc đầy đủ — đối soát phải phân biệt rõ luồng tiền của cuốc hoàn tất và cuốc hủy có phí bồi thường, không gộp chung logic tính hoa hồng như cuốc bình thường.
- Giá cuốc thực tế cuối cùng có thể khác giá ước tính ban đầu hiển thị cho khách (do đổi lộ trình giữa đường, chờ đợi phát sinh, tranh chấp về quãng đường) — đối soát phải dùng đúng giá đã chốt cuối cùng làm cơ sở tính hoa hồng và tiền tài xế, xử lý được trường hợp giá bị điều chỉnh lại sau khi cuốc đã kết thúc và tiền đã tạm ghi nhận.
- Tài xế nhận tiền mặt trực tiếp từ khách cho một số cuốc trong khi hoa hồng nền tảng vẫn phải được thu lại từ tài xế theo cách khác (trừ vào ví/đợt thanh toán sau) — đối soát phải theo dõi riêng luồng tiền mặt này để không nhầm là "tiền chưa về" khi thực chất khách đã trả trực tiếp cho tài xế.
- Khi có tranh chấp giá giữa khách và tài xế được xử lý sau khi cuốc đã kết thúc và tiền đã được chia, khoản điều chỉnh phát sinh sau phải được đối soát lại đúng cho cả ba bên mà không làm sai lệch báo cáo hoa hồng của các cuốc khác không liên quan trong cùng kỳ.
- Với tài xế có khối lượng cuốc lớn mỗi ngày, tổng tiền tài xế nhận được tổng hợp theo đợt thanh toán định kỳ phải khớp chính xác với tổng từng cuốc riêng lẻ cộng lại trừ các khoản điều chỉnh phát sinh trong kỳ — cần cơ chế đối soát ở cả mức chi tiết từng cuốc và mức tổng hợp theo đợt để phát hiện sai lệch dù nhỏ không bị pha loãng/che khuất khi chỉ nhìn tổng số cuối kỳ.
