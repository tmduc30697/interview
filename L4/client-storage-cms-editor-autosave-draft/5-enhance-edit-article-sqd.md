# Enhance sequence — Edit article (debounce autosave vào IndexedDB, đo ngưỡng gây giật)

Đây là **enhance**, cùng flow "edit-article" như ở base nhưng thay đổi hoàn toàn cơ chế autosave: thay vì gọi thẳng server định kỳ, mỗi lần ngừng gõ 2-3 giây mới serialize và ghi 1 bản draft vào IndexedDB (debounce), đồng thời đo thời gian serialize/ghi theo độ dài nội dung để xác định ngưỡng bắt đầu gây giật khi gõ. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant Debounce as Debounce Timer
    participant IDB as IndexedDB (LOCAL_DRAFT)
    participant Metric as TYPING_PERF_METRIC

    User->>Editor: Gõ liên tục (keystroke)
    loop Mỗi keystroke
        Editor->>Debounce: Reset timer debounce 2-3 giây
    end
    Note over Debounce: Chỉ khi người dùng NGỪNG gõ đủ 2-3 giây, timer mới bắn
    Debounce->>Editor: Timer hết hạn, kích hoạt autosave cục bộ
    Editor->>Editor: Serialize nội dung hiện tại, đo serialize_ms
    Editor->>IDB: Ghi LOCAL_DRAFT mới (content_snapshot, content_length, updated_at_local), đo write_ms
    IDB-->>Editor: Ghi thành công
    Editor->>Metric: Ghi nhận content_length_bucket, serialize_ms, write_ms

    alt Tổng serialize_ms + write_ms vượt ngưỡng gây giật (ví dụ nội dung hàng chục nghìn từ)
        Metric->>Metric: Đánh dấu jank_detected=true cho bucket độ dài này
        Note over Metric: Dữ liệu này dùng để xác định ngưỡng nên tối ưu, ví dụ serialize từng phần thay vì toàn bộ document
    else Trong ngưỡng bình thường
        Metric->>Metric: jank_detected=false
    end
```
