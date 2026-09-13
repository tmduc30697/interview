# ERD - Base (trước khi xử lý reservation đúng cấp biến thể)

Đây là **base**: mô hình dữ liệu tối thiểu của một sàn thời trang, mỗi sản phẩm có nhiều biến thể (size, màu), mỗi biến thể có tồn kho riêng. Chỉ giữ `Product`, `Variant`, `Customer`, `Reservation` - đủ để thấy vấn đề gốc: `Variant.quantity` là một cột đơn giản không có khóa nguyên tử theo tổ hợp size/màu, và không có gì phân biệt trạng thái "còn hàng" hiển thị tổng với tồn kho khả dụng thực của từng biến thể.

```mermaid
erDiagram
    PRODUCT ||--o{ VARIANT : has
    VARIANT ||--o{ RESERVATION : reserved_in
    CUSTOMER ||--o{ RESERVATION : creates

    PRODUCT {
        string id PK
        string title
    }

    VARIANT {
        string id PK
        string product_id FK
        string size
        string color
        int quantity
    }

    CUSTOMER {
        string id PK
        string name
    }

    RESERVATION {
        string id PK
        string variant_id FK
        string customer_id FK
        int quantity
        string status
        datetime created_at
    }
```
