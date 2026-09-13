# Base ERD — Marketplace trước khi siết cách ly seller ở tầng dữ liệu

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** siết cách ly. Đề bài đối chiếu trực tiếp với việc hiện tại chỉ "ẩn nút bấm trên giao diện" — nên base được suy luận là một marketplace đã có đủ: seller, sản phẩm, đơn hàng có thể gộp nhiều seller trong cùng giỏ hàng (`ORDER_ITEM.seller_id` đã tồn tại vì đây là cấu trúc tự nhiên của giỏ hàng đa seller, không phải điều enhance mới thêm), và một báo cáo benchmark ngành đã có sẵn nhưng tính trung bình ngây thơ không xét cỡ mẫu. Base **chưa có** cơ chế enforce `seller_id` ở tầng data/query layer (chỉ dựa vào UI ẩn), **chưa có** ngưỡng mẫu tối thiểu cho benchmark, **chưa có** quy trình offboarding seller giữ lịch sử nhưng thu hồi quyền truy cập, và **chưa có** log tra cứu đơn hàng có phạm vi giới hạn cho nhân viên hỗ trợ — những phần này là enhance.

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "ordered as"
    SELLER ||--o{ ORDER_ITEM : sells
    INDUSTRY_BENCHMARK_REPORT ||--o{ ORDER_ITEM : "computed from (cùng category)"

    SELLER {
        string id PK
        string name
        string status "active"
        datetime joined_at
    }
    BUYER {
        string id PK
        string name
        string email
    }
    PRODUCT {
        string id PK
        string seller_id FK
        string category
        string title
        decimal price
        int stock
    }
    ORDER {
        string id PK
        string buyer_id FK
        datetime created_at
        string status
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        string seller_id FK "denormalized từ product, phục vụ giỏ hàng đa seller"
        int quantity
        decimal unit_price
        string shipping_status
    }
    INDUSTRY_BENCHMARK_REPORT {
        string id PK
        string category
        decimal avg_revenue "tính trung bình ngây thơ, không xét cỡ mẫu"
        datetime computed_at
    }
```
