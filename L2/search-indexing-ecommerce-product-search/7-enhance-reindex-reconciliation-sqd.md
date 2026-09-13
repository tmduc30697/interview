# Sequence Diagram — Enhance: Reindex Reconciliation

Đây là **enhance**, flow hoàn toàn mới: job đối soát định kỳ phát hiện index bị lệch so với database nguồn (do lỗi đồng bộ), tự phục hồi bằng full/partial reindex sang index song song rồi swap alias, không ảnh hưởng tìm kiếm đang chạy.

```mermaid
sequenceDiagram
    participant Job as Reconciliation Job
    participant DB as Product Database
    participant Index as Active Product Index
    participant Standby as Standby Index
    participant Alias as Index Alias Service

    Job->>DB: Lấy mẫu/toàn bộ dữ liệu sản phẩm nguồn
    Job->>Index: Lấy dữ liệu tương ứng đang có trong index
    Job->>Job: So sánh, phát hiện các bản ghi lệch

    alt Có lệch phát hiện
        Job->>Standby: Build lại dữ liệu (full hoặc partial reindex) trên index song song
        Standby-->>Job: Reindex hoàn tất, đã khớp với DB nguồn
        Job->>Alias: Swap alias trỏ từ Active Index sang Standby Index
        Alias-->>Job: Alias switched, tìm kiếm chuyển sang index mới ngay lập tức
        Note over Index,Standby: Trong suốt quá trình reindex, tìm kiếm vẫn phục vụ trên index cũ, không downtime
    else Không lệch
        Job->>Job: Ghi nhận reconciliation run thành công, không cần reindex
    end
```
