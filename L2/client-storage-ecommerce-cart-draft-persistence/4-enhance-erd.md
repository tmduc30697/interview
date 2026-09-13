# Enhance ERD — Giỏ hàng và checkout draft qua localStorage

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (CART/CART_ITEM chỉ có ý nghĩa cho user đã đăng nhập, guest cart chỉ ở bộ nhớ), phần enhance thêm 5 nhóm entity mới ở phía client, ứng trực tiếp với các yêu cầu trong đề bài:

- `LOCAL_CART_ITEM` (mới, localStorage) — giỏ hàng guest được lưu bền thay vì chỉ ở bộ nhớ, kèm `price_snapshot` tại thời điểm thêm — nền cho yêu cầu 1 và 2.
- `CART_MERGE_LOG` (mới) — ghi lại quyết định merge giữa giỏ local và giỏ server khi đăng nhập (cộng dồn, giữ số lượng lớn hơn, hoặc hỏi user) — yêu cầu 1.
- `PRICE_REVALIDATION` (mới) — kết quả so sánh `price_snapshot` cục bộ với giá/tồn kho thật tại thời điểm checkout — yêu cầu 2.
- `CART_SYNC_EVENT` (mới) — sự kiện phát qua `storage`/BroadcastChannel khi 1 tab add/remove item, để tab khác cập nhật badge mà không ghi đè thay đổi của nhau — yêu cầu 3.
- `STORAGE_FALLBACK_STATE` (mới) — trạng thái localStorage có khả dụng không (quota exceeded hoặc bị chặn), có đang fallback in-memory không — yêu cầu 4.
- `CHECKOUT_DRAFT_LOCAL` (mới, localStorage) — form checkout dở dang (địa chỉ, note) kèm `expires_at` để tự hết hạn — yêu cầu 5.

```mermaid
erDiagram
    USER ||--o| CART : "has (khi đã đăng nhập)"
    CART ||--o{ CART_ITEM : contains
    PRODUCT ||--o{ CART_ITEM : "referenced by"
    PRODUCT ||--o{ LOCAL_CART_ITEM : "referenced by"
    LOCAL_CART_ITEM ||--o{ PRICE_REVALIDATION : "checked before checkout"
    LOCAL_CART_ITEM ||--o{ CART_SYNC_EVENT : "triggers"
    USER ||--o{ CART_MERGE_LOG : "merge history on login"
    USER ||--o| CHECKOUT_DRAFT_LOCAL : "has draft (localStorage)"
    USER ||--o| STORAGE_FALLBACK_STATE : "has per browser session"

    USER {
        string id PK
        string email
    }
    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
    }
    CART {
        string id PK
        string user_id FK
        string status "active | ordered"
    }
    CART_ITEM {
        string id PK
        string cart_id FK
        string product_id FK
        int quantity
        decimal price_at_add
    }
    LOCAL_CART_ITEM {
        string id PK
        string product_id FK
        int quantity
        decimal price_snapshot
        datetime added_at
        datetime updated_at
    }
    CART_MERGE_LOG {
        string id PK
        string user_id FK
        string product_id FK
        int local_quantity
        int server_quantity
        string resolution "sum | keep_max | asked_user"
        datetime merged_at
    }
    PRICE_REVALIDATION {
        string id PK
        string local_cart_item_id FK
        decimal price_snapshot
        decimal current_price
        int current_stock
        string status "unchanged | price_changed | out_of_stock"
        datetime checked_at
    }
    CART_SYNC_EVENT {
        string id PK
        string origin_tab_id
        string product_id FK
        string action "add | remove | update_qty"
        int quantity
        datetime broadcast_at
    }
    STORAGE_FALLBACK_STATE {
        string id PK
        string user_id FK
        string session_id
        boolean local_storage_available
        boolean fallback_in_memory_active
        datetime detected_at
    }
    CHECKOUT_DRAFT_LOCAL {
        string id PK
        string user_id FK
        string address
        string note
        datetime saved_at
        datetime expires_at
    }
```
