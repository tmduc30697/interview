# ERD - Base (trước khi có reservation atomic nhiều kho)

Đây là **base**: mô hình dữ liệu của sàn e-commerce nhiều kho, nơi tồn kho đã có khái niệm "soft reserve 15 phút" nhưng được cài đặt đơn giản - `reserved` chỉ là một cột số nguyên tăng/giảm trực tiếp trên `Inventory`, không có bảng reservation riêng để truy ngược ai đang giữ, giữ từ khi nào, hết hạn lúc nào. `CartItem` chỉ lưu `reserved_at` để một cron job dựa vào đó blind-decrement `reserved` sau 15 phút.

```mermaid
erDiagram
    CUSTOMER ||--o{ CART : owns
    CUSTOMER ||--o{ CART_ITEM : reserves_via
    CART ||--o{ CART_ITEM : contains
    WAREHOUSE ||--o{ INVENTORY : stocks
    PRODUCT ||--o{ INVENTORY : tracked_in
    INVENTORY ||--o{ CART_ITEM : reserved_from

    CUSTOMER {
        string id PK
        string region
    }

    WAREHOUSE {
        string id PK
        string name
        string region
    }

    PRODUCT {
        string id PK
        string title
    }

    INVENTORY {
        string id PK
        string product_id FK
        string warehouse_id FK
        int available
        int reserved
    }

    CART {
        string id PK
        string customer_id FK
    }

    CART_ITEM {
        string id PK
        string cart_id FK
        string product_id FK
        string warehouse_id FK
        int quantity
        datetime reserved_at
    }
```
