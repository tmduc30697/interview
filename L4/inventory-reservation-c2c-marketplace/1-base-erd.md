# ERD - Base (trước khi xử lý concurrency cho reservation C2C)

Đây là **base**: mô hình dữ liệu tối thiểu của một marketplace C2C, nơi seller đăng sản phẩm với số lượng, buyer đặt mua để giữ chỗ. Chỉ giữ `Seller`, `Buyer`, `Product`, `Reservation` - đủ để thấy vấn đề: `Product.quantity` là một con số đơn giản, còn `Reservation` chưa có TTL rõ ràng hay cơ chế atomic, chưa có audit log - đúng những gì enhance sẽ bổ sung.

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ RESERVATION : creates
    PRODUCT ||--o{ RESERVATION : reserved_in

    SELLER {
        string id PK
        string name
    }

    BUYER {
        string id PK
        string name
    }

    PRODUCT {
        string id PK
        string seller_id FK
        string title
        int quantity
        string status
        datetime created_at
    }

    RESERVATION {
        string id PK
        string product_id FK
        string buyer_id FK
        int quantity
        string status
        datetime created_at
    }
```
