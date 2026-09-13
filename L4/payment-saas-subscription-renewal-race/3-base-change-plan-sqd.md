# Base sequence — Khách tự đổi gói/hủy gói qua UI (ghi đè trực tiếp)

Đây là **base**, flow "Khách đổi gói/hủy gói" ở trạng thái hiện tại — khách thao tác qua UI, hệ thống ghi đè thẳng `plan_id`/`status` của subscription ngay lập tức, không biết và không quan tâm job renewal có đang xử lý song song subscription này hay không. Flow này liên quan mật thiết tới enhance vì đây chính là phía còn lại của race điều kiện mà đề bài mô tả ở yêu cầu 1 và 2.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Subscription Service
    participant DB as SUBSCRIPTION store

    Customer->>App: Bấm "Hủy subscription" (hoặc đổi gói Pro sang Basic)
    App->>DB: UPDATE SUBSCRIPTION SET status=cancelled (hoặc plan_id=Basic) ngay lập tức
    App-->>Customer: "Đã hủy thành công" (hoặc "Đã đổi gói thành công")

    Note over App,DB: Không có bước nào kiểm tra job renewal có đang giữa chừng charge thẻ cho subscription này hay không, dẫn tới 2 kịch bản race ở yêu cầu 1 và 2 của đề bài
```
