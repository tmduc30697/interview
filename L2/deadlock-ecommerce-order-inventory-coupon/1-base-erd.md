# Base ERD — Checkout chưa chuẩn hóa thứ tự lock

Đây là **base**: trạng thái hệ thống e-commerce *trước khi* áp dụng thứ tự lock cố định toàn cục. Suy luận từ đề bài, base đã có `PRODUCT`/`INVENTORY` (tồn kho theo sản phẩm), `ORDER`/`ORDER_ITEM` (đơn hàng và các dòng sản phẩm trong đơn), và `COUPON` (mã giảm giá dùng 1 lần, giới hạn số lượt). `ORDER_ITEM` có trường `added_sequence` ghi lại đúng thứ tự khách thêm sản phẩm vào giỏ — và trong base, code trừ tồn kho lại đi theo đúng thứ tự này (không chuẩn hóa), chính là nguồn gốc rủi ro deadlock nêu ở đề bài.

```mermaid
erDiagram
    PRODUCT ||--|| INVENTORY : "có tồn kho"
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "được đặt trong"
    ORDER }o--o| COUPON : "áp dụng (nếu có)"

    PRODUCT {
        string id PK
        string name
    }
    INVENTORY {
        string product_id PK
        int quantity
    }
    ORDER {
        string id PK
        string coupon_id FK "nullable"
        string status "pending|success|failed"
        datetime created_at
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        int quantity
        int added_sequence "thứ tự khách thêm vào giỏ, chưa chuẩn hóa"
    }
    COUPON {
        string id PK
        string code
        int remaining_uses
    }
```
