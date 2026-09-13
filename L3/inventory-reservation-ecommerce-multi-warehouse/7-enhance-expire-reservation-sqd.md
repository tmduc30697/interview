# Sequence - Enhance - Flow "expire-reservation"

Đây là **enhance**, cùng flow `expire-reservation` như ở base nhưng thay blind-decrement bằng kiểm tra trạng thái tường minh trên bảng `Reservation`, đáp ứng yêu cầu 3: transaction thanh toán phải kiểm tra reservation vẫn còn `status = active` ngay trước khi trừ tồn kho thật, nếu đã bị job expire chuyển sang `expired` thì từ chối thanh toán rõ ràng và không charge tiền - thay vì tình trạng mơ hồ như ở base.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)
    participant ExpireJob as Cron Expire Job

    Note over Inventory: Reservation.status = active, expires_at đã qua 15 phút

    par Request thanh toán tới gần như đồng thời
        Customer->>App: xác nhận thanh toán
        App->>Inventory: BEGIN transaction, khóa dòng Reservation (SELECT FOR UPDATE)
        App-->>App: kiểm tra Reservation.status = active và chưa bị expire job xử lý
        alt Reservation vẫn active tại thời điểm khóa
            App->>Inventory: trừ tồn kho thật, available -= 1, reserved -= 1
            App->>App: Reservation.status = consumed
            App-->>Customer: thanh toán thành công, charge tiền
        else Reservation đã bị expire job chuyển sang expired trước khi khóa được
            App-->>Customer: từ chối thanh toán rõ ràng, reservation đã hết hạn, không charge tiền
        end
    and Cron job chạy đúng lúc đó
        ExpireJob-->>ExpireJob: quét Reservation có expires_at đã qua và status = active
        ExpireJob->>Inventory: UPDATE reservation SET status = expired WHERE status = active
        ExpireJob->>Inventory: reserved -= 1 (chỉ khi update status ở trên thành công)
    end

    Note over App,ExpireJob: Chỉ 1 trong 2 nhánh thắng nhờ khóa dòng Reservation,\nkhông bao giờ vừa charge tiền vừa đã giải phóng tồn kho
```
