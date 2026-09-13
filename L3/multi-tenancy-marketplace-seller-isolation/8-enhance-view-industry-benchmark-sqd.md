# Enhance sequence — Xem benchmark ngành (chặn suy luận ngược khi cỡ mẫu nhỏ)

Đây là **enhance**, cùng flow "Xem báo cáo so sánh hiệu suất với trung bình ngành" đã có ở base nhưng nay thay đổi theo yêu cầu 3 của đề bài: trước khi trả số liệu so sánh cụ thể, hệ thống kiểm tra `seller_count_in_sample` so với `min_sample_size`, nếu ngành hàng quá ít seller thì không tiết lộ số liệu chi tiết để tránh suy luận ngược ra đối thủ cụ thể.

```mermaid
sequenceDiagram
    actor SellerA as Seller A (ngành hàng chỉ có 2 seller)
    participant App as Benchmark Service
    participant DB as INDUSTRY_BENCHMARK_REPORT

    SellerA->>App: Xem so sánh doanh thu với "mức trung bình ngành"
    App->>DB: Lấy avg_revenue, seller_count_in_sample, min_sample_size của category
    DB-->>App: avg_revenue, seller_count_in_sample=2, min_sample_size=5

    alt seller_count_in_sample >= min_sample_size
        App-->>SellerA: Hiển thị số liệu so sánh cụ thể
    else seller_count_in_sample < min_sample_size
        App->>DB: Đánh dấu suppressed=true cho report này
        App-->>SellerA: Hiển thị thông báo chung "Chưa đủ dữ liệu ngành để so sánh chi tiết", không lộ avg_revenue cụ thể
        Note over App: Với cỡ mẫu quá nhỏ, dù chỉ đưa ra 1 con số trung bình cũng đủ để suy luận ngược ra đối thủ duy nhất còn lại — nên phải generalize hoặc gộp thêm category liên quan thay vì hiển thị số cụ thể
    end
```
