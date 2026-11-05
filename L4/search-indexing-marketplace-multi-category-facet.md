# Marketplace đa category với facet lọc khác biệt hoàn toàn

**Hệ thống:** Một marketplace bán hàng đa category hoàn toàn khác biệt (điện tử, thời trang, nội thất...), mỗi category có bộ thuộc tính/facet lọc riêng.

**Vai trò của flow:** Lập chỉ mục sản phẩm từ nhiều category với schema thuộc tính khác nhau, phục vụ tìm kiếm xuyên category và autocomplete gợi ý đúng theo ngữ cảnh category đang tìm.

**Yêu cầu cụ thể:**
- Mỗi category có bộ facet lọc riêng (điện tử có "dung lượng RAM", thời trang có "size/màu", nội thất có "chất liệu") — index phải lưu và truy vấn hiệu quả với schema thuộc tính động theo category mà không phải tạo index riêng biệt cho từng category gây khó khăn khi tìm xuyên category.
- Khi người dùng gõ một từ khóa mơ hồ có thể thuộc nhiều category (ví dụ "bàn" có thể là bàn ăn nội thất hoặc liên quan đến bàn phím máy tính), autocomplete phải suy luận được ngữ cảnh category phù hợp dựa trên hành vi tìm kiếm/lịch sử, tránh gợi ý lẫn lộn không liên quan gây khó chịu.
- Khi sản phẩm được người bán đổi category sau khi đã đăng, index phải cập nhật đồng thời cả danh mục lẫn bộ facet thuộc tính tương ứng, đảm bảo sản phẩm không còn xuất hiện ở bộ lọc facet của category cũ không còn phù hợp.
- Kết quả tìm kiếm xuyên category (ví dụ tìm "quà tặng sinh nhật" trả về cả điện tử, thời trang, đồ chơi) phải xếp hạng công bằng giữa các category khác nhau về bản chất — độ khớp từ khóa của một sản phẩm thời trang không thể so trực tiếp với một sản phẩm điện tử theo cùng công thức điểm số mà không tính đến đặc thù từng category.
- Khi thêm một category hoàn toàn mới vào hệ thống, việc bổ sung schema/facet mới cho category đó không được yêu cầu reindex toàn bộ dữ liệu các category hiện có đang phục vụ tìm kiếm sống.
