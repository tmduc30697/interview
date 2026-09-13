# Sequence Diagram - Enhance: Bảo trì index khi sự kiện viral

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base. Flow này giải quyết vấn đề đề bài nêu: ở các sự kiện viral, lượng insert đột biến vào `posts` khiến phần cuối index theo `created_at` bị ghi liên tục dẫn tới page split và phân mảnh, nên cần tuning `fillfactor` và/hoặc lịch bảo trì/rebuild index định kỳ.

```mermaid
sequenceDiagram
    participant Monitor as Ops Monitor
    participant DB as Postgres (posts + index)
    participant Job as Index Maintenance Job

    Note over DB: Su kien viral, insert dot bien vao posts
    Note over DB: Phan cuoi index composite (author_id, created_at) bi ghi lien tuc
    DB->>Monitor: canh bao bloat/phan manh tang cao tren index
    Monitor->>Job: kich hoat quy trinh bao tri
    Job->>DB: kiem tra fillfactor hien tai cua bang posts
    alt fillfactor chua toi uu cho khoi luong ghi lon
        Job->>DB: ALTER TABLE posts SET (fillfactor = 70)
        Note over DB: Chua san khoang trong tren moi trang de giam page split khi insert/update lien tuc
    end
    Job->>DB: REINDEX CONCURRENTLY hoac chay theo lich rebuild index dinh ky
    DB-->>Job: index da duoc don dep, giam phan manh
    Job-->>Monitor: bao cao hoan tat bao tri
```
