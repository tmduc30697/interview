# Enhance sequence — Test giả lập checkout hoán vị giỏ hàng ngẫu nhiên, có/không coupon

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 5 của đề bài: giả lập nhiều đơn hàng với thứ tự sản phẩm trong giỏ được hoán vị ngẫu nhiên (bao gồm cả có/không dùng coupon), xác nhận không transaction nào treo quá timeout và invariant tồn kho + số lượt dùng coupon còn lại luôn đúng sau khi mọi retry hoàn tất.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant Orders as N đơn hàng (giỏ hàng hoán vị ngẫu nhiên, có/không coupon)
    participant DB as Database (lock order + retry + timeout đã áp dụng)
    participant Inv as INVENTORY (nhiều sản phẩm)
    participant Cpn as COUPON

    Test->>Inv: Đọc snapshot tồn kho ban đầu của từng sản phẩm
    Test->>Cpn: Đọc remaining_uses ban đầu của coupon
    Test->>Test: Tính expected_deduction dựa trên toàn bộ đơn hàng dự kiến thành công

    Test->>Orders: Khởi chạy đồng thời N đơn hàng, mỗi đơn có thứ tự sản phẩm trong giỏ ngẫu nhiên
    par Chạy song song trong vài giây cao điểm
        Orders->>DB: Đơn hàng có coupon (retry tối đa 3 lần, timeout 2 giây)
    and
        Orders->>DB: Đơn hàng không coupon (retry tối đa 3 lần, timeout 2 giây)
    end
    DB-->>Orders: Mỗi đơn hàng hoặc commit thành công trong timeout, hoặc bị hủy rõ ràng với lỗi "thử lại sau"

    Test->>Test: assert không đơn hàng nào có tổng thời gian xử lý > timeout_ms=2000
    Test->>Inv: Đọc tồn kho sau khi toàn bộ retry hoàn tất
    Test->>Cpn: Đọc remaining_uses sau khi toàn bộ retry hoàn tất
    Test->>Test: assert tồn kho == tồn kho ban đầu - tổng số lượng đã trừ bởi các đơn thành công
    Test->>Test: assert remaining_uses == remaining_uses ban đầu - số đơn thành công có dùng coupon

    Note over Test,DB: Vì thứ tự lock đã chuẩn hóa toàn cục, kết quả invariant không phụ thuộc vào thứ tự hoán vị giỏ hàng hay đơn nào chạy trước
```
