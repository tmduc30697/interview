# Sequence Diagram — Enhance: Upload and Publish Video

Đây là **enhance**, flow đã thay đổi so với base ([2-base-upload-and-publish-video-sqd.md](2-base-upload-and-publish-video-sqd.md)): upload nay chia chunk (xem chi tiết resume ở [5-enhance-resumable-chunked-upload-sqd.md](5-enhance-resumable-chunked-upload-sqd.md)), transcode nhiều độ phân giải song song không chặn nhau, và video publish ngay khi có rendition đầu tiên thay vì chờ toàn bộ như base.

```mermaid
sequenceDiagram
    actor User
    participant VideoSvc as Video Service
    participant Storage as Video Storage
    participant Queue as Transcode Queue
    participant Transcoder1 as Worker (1080p)
    participant Transcoder2 as Worker (720p)
    participant Transcoder3 as Worker (480p)

    User->>VideoSvc: Upload video theo chunk (xem flow resumable riêng)
    VideoSvc->>Storage: Ghép chunk thành raw file hoàn chỉnh
    VideoSvc->>Queue: Enqueue transcode job cho 3 độ phân giải

    par transcode song song, không block nhau
        Queue->>Transcoder1: Transcode 1080p
    and
        Queue->>Transcoder2: Transcode 720p
    and
        Queue->>Transcoder3: Transcode 480p
    end

    Transcoder3-->>VideoSvc: 480p ready trước (ít tài nguyên nhất)
    VideoSvc->>VideoSvc: Publish video ngay khi có 1 rendition sẵn sàng
    VideoSvc-->>User: Video public, xem được ở 480p

    Transcoder2-->>VideoSvc: 720p ready
    Transcoder1-->>VideoSvc: 1080p ready (có thể fail riêng, không ảnh hưởng rendition khác)
    VideoSvc-->>User: Các độ phân giải còn lại hoàn thành dần
```
