# Sequence Diagram — Enhance: Resumable Chunked Upload

Đây là **enhance**, flow mới hoàn toàn phục vụ yêu cầu upload file lớn (nhiều GB) bằng chunked/resumable upload, tiếp tục được nếu mất mạng giữa chừng — chưa tồn tại ở base vì base upload nguyên khối.

```mermaid
sequenceDiagram
    actor User
    participant VideoSvc as Video Service
    participant Storage as Video Storage

    User->>VideoSvc: Bắt đầu upload, chia file thành UPLOAD_CHUNK
    loop mỗi chunk
        User->>VideoSvc: Gửi chunk (chunk_index, data)
        VideoSvc->>Storage: Lưu chunk, đánh dấu status = uploaded
        VideoSvc-->>User: Ack chunk
    end

    Note over User,VideoSvc: Mất mạng giữa chừng ở chunk N

    User->>VideoSvc: Kết nối lại, hỏi trạng thái upload
    VideoSvc-->>User: Danh sách chunk đã upload thành công (1..N-1)
    User->>VideoSvc: Tiếp tục gửi từ chunk N, không upload lại từ đầu

    VideoSvc->>Storage: Nhận đủ chunk cuối cùng
    VideoSvc->>Storage: Ghép các chunk thành raw file hoàn chỉnh theo đúng thứ tự
    VideoSvc-->>User: Upload hoàn tất
```
