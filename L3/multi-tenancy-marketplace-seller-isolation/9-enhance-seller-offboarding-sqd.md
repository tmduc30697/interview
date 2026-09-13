# Enhance sequence — Seller offboarding (thu hồi quyền, giữ lịch sử, không lộ qua báo cáo tổng hợp)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có quy trình xử lý khi seller ngừng hoạt động, chỉ có `status = active` cố định. Đáp ứng yêu cầu 4 của đề bài: dữ liệu lịch sử đơn hàng vẫn giữ cho hỗ trợ khách hàng/pháp lý, seller đã rời sàn mất quyền truy cập, và các seller khác không vô tình thấy dữ liệu của seller đã rời qua báo cáo tổng hợp toàn nền tảng.

```mermaid
sequenceDiagram
    actor Ops as Vận hành nền tảng
    participant App as Seller Management Service
    participant SellerDB as SELLER store
    participant Event as SELLER_OFFBOARDING_EVENT
    participant OrderDB as ORDER / ORDER_ITEM store
    participant Bench as INDUSTRY_BENCHMARK_REPORT
    actor OffSeller as Seller đã rời sàn
    actor OtherSeller as Seller khác vẫn hoạt động

    Ops->>App: Khóa/offboard Seller X (vi phạm chính sách hoặc tự nguyện)
    App->>SellerDB: Cập nhật SELLER(X).status=offboarded, deactivated_at=now
    App->>Event: Tạo SELLER_OFFBOARDING_EVENT(seller_id=X, reason, data_retention_note)
    Note over OrderDB: ORDER/ORDER_ITEM lịch sử của Seller X không bị xóa, vẫn giữ nguyên cho mục đích hỗ trợ khách mua và pháp lý

    OffSeller->>App: Cố đăng nhập lại dashboard
    App->>SellerDB: Kiểm tra status
    SellerDB-->>App: status=offboarded
    App-->>OffSeller: Từ chối truy cập, tài khoản đã bị thu hồi quyền

    OtherSeller->>App: Xem benchmark ngành (category có cả Seller X trong lịch sử)
    App->>Bench: Tính avg_revenue có gộp dữ liệu lịch sử của Seller X, kiểm tra seller_count_in_sample >= min_sample_size như flow benchmark
    Bench-->>App: Kết quả đã qua kiểm tra ngưỡng mẫu tối thiểu
    App-->>OtherSeller: Hiển thị số liệu tổng hợp an toàn, không có cách nào tách riêng ra đóng góp cụ thể của Seller X
```
