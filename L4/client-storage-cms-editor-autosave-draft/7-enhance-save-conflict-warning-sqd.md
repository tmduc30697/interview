# Enhance sequence — Cảnh báo xung đột khi server đã có bản mới hơn từ tab/máy khác

Đây là **enhance**, flow hoàn toàn mới, xử lý tình huống mở cùng 1 bài viết ở 2 tab hoặc 2 máy khác nhau. Vì draft cục bộ trong IndexedDB chỉ có ý nghĩa cục bộ, không đồng bộ qua lại giữa các máy, nên trước khi lưu đè lên server hệ thống phải kiểm tra xem server đã bị sửa từ nơi khác mới hơn baseline mà tab hiện tại đang biết hay chưa, và cảnh báo rõ nguy cơ mất nội dung thay vì im lặng ghi đè. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài (Tab A)
    participant EditorA as Editor Tab A
    participant IDB as IndexedDB (LOCAL_DRAFT, cục bộ theo máy)
    participant API as Article API
    actor OtherUser as Người soạn bài (Tab B / máy khác)

    Note over EditorA: Tab A đang giữ baseline updated_at của bài viết từ lúc mở
    OtherUser->>API: Lưu thay đổi từ Tab B / máy khác, ARTICLE.updated_at được cập nhật mới hơn

    User->>EditorA: Tiếp tục gõ ở Tab A, tới lúc autosave/lưu
    EditorA->>IDB: Ghi LOCAL_DRAFT mới (chỉ cục bộ trên máy này)
    EditorA->>API: Trước khi lưu lên server, kiểm tra ARTICLE.updated_at hiện tại
    API-->>EditorA: Trả về updated_at mới hơn baseline mà Tab A đang có

    alt updated_at server mới hơn baseline của Tab A
        EditorA-->>User: Cảnh báo rõ, bài viết đã được sửa ở nơi khác, lưu đè có thể làm mất nội dung đó
        alt User chọn vẫn lưu đè
            EditorA->>API: Ghi đè content từ Tab A lên server
        else User chọn xem lại/hủy lưu
            EditorA-->>User: Không lưu, để user tự quyết định merge thủ công
        end
    else Không có xung đột, server vẫn là baseline cũ
        EditorA->>API: Lưu bình thường, không cần cảnh báo
    end
```
