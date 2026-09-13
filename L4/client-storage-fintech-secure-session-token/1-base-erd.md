# Base ERD — Lưu token phía client trước khi siết bảo mật

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống **trước khi** áp các yêu cầu bảo mật nghiêm ngặt. Đề bài ngầm định hệ thống ngân hàng số hiện đã có đăng nhập và lưu token phía client theo cách đơn giản/naive: cả access token lẫn refresh token đều được lưu chung vào localStorage sau khi đăng nhập, không phân loại theo mức độ nhạy cảm, không có idle timeout, không có đồng bộ đăng xuất giữa các tab, chưa xử lý bfcache. Đây chính là những vấn đề mà enhance sẽ giải quyết.

```mermaid
erDiagram
    USER ||--o| CLIENT_LOCAL_SESSION : "stores tokens in localStorage"

    USER {
        string id PK
        string email
    }
    CLIENT_LOCAL_SESSION {
        string id PK
        string user_id FK
        string access_token "lưu trong localStorage, mọi script trên trang đều đọc được"
        string refresh_token "lưu trong localStorage, cùng rủi ro như access_token"
        datetime stored_at
    }
```
