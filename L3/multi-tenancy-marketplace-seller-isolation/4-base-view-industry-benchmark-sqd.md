# Base sequence — Xem benchmark ngành (trung bình ngây thơ, có thể suy luận ngược)

Đây là **base**, flow "Xem báo cáo so sánh hiệu suất với trung bình ngành" ở trạng thái hiện tại — tính trung bình doanh thu toàn bộ seller trong category mà không xét số lượng seller tham gia. Flow này liên quan mật thiết tới enhance vì yêu cầu 3 của đề bài chính là chặn kiểu suy luận ngược này.

```mermaid
sequenceDiagram
    actor SellerA as Seller A (ngành hàng chỉ có 2 seller)
    participant App as Benchmark Service
    participant DB as INDUSTRY_BENCHMARK_REPORT

    SellerA->>App: Xem so sánh doanh thu với "mức trung bình ngành"
    App->>DB: Lấy avg_revenue của category (tính trên toàn bộ seller trong category, không xét cỡ mẫu)
    DB-->>App: avg_revenue (tính trên đúng 2 seller: A và B)
    App-->>SellerA: Hiển thị "Doanh thu trung bình ngành: X"
    Note over SellerA: Seller A tự biết doanh thu của mình, suy ra ngay doanh thu Seller B = 2*X - doanh_thu_A — số liệu đối thủ bị lộ hoàn toàn
```
