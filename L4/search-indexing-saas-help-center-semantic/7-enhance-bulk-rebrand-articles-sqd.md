# Sequence Diagram — Enhance: Bulk Rebrand Articles

Đây là **enhance**, flow hoàn toàn mới: khi đội docs sửa hàng loạt bài viết cùng lúc (đổi tên tính năng, rebrand), pipeline index phải cập nhật atomic, không để khoảng thời gian index ở trạng thái nửa cũ nửa mới hiển thị lẫn lộn cho các khách hàng khác nhau.

```mermaid
sequenceDiagram
    actor DocsTeam as Đội Docs
    participant HelpCenter as Help Center App
    participant DB as Articles Database
    participant Indexer as Index Sync Worker
    participant Standby as Standby Index
    participant Alias as Index Alias Service
    participant Active as Active Article Index

    DocsTeam->>HelpCenter: Thực hiện rebrand hàng loạt (đổi tên tính năng trên nhiều bài)
    HelpCenter->>DB: Batch UPDATE articles SET content
    DB-->>HelpCenter: Batch updated

    HelpCenter->>Indexer: Trigger REINDEX_BATCH (bulk mode)
    Indexer->>Standby: Build lại toàn bộ document liên quan trên index song song (không đụng index đang phục vụ)
    Standby-->>Indexer: Reindex hoàn tất

    Indexer->>Alias: Swap alias từ Active Index sang Standby Index (atomic)
    Alias-->>Indexer: Switched

    Note over Active,Standby: Khách hàng luôn thấy 1 trong 2 trạng thái trọn vẹn (toàn bộ cũ hoặc toàn bộ mới), không có lúc nửa cũ nửa mới
    Indexer-->>HelpCenter: Bulk rebrand indexed thành công
```
