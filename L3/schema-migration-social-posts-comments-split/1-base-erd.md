# ERD — Base (trước khi tách bảng comments)

Đây là **base**: mô hình dữ liệu suy luận cho mạng xã hội *trước khi* tách comment. Đề bài giả định đã có `USER` đăng `POST`, và comment được lưu nested/JSON ngay trong 1 cột của `POST` — nếu không có sẵn cột JSON đó thì yêu cầu "tách sang bảng comments quan hệ riêng" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ đăng bài/bình luận, không suy diễn thêm like/share/notification không liên quan.

```mermaid
erDiagram
    USER ||--o{ POST : creates

    USER {
        string user_id PK
        string username
        string email
    }

    POST {
        string post_id PK
        string user_id FK
        string content
        string comments_json
        datetime created_at
    }
```
