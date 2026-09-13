# Sequence Diagram — Base: Upload and Publish Video

Đây là **base**, flow upload và publish đơn giản (nguyên khối, 1 độ phân giải, publish khi xong toàn bộ) — tiền đề bắt buộc mà enhance sẽ nâng cấp thành resumable/multi-resolution/progressive.

```mermaid
sequenceDiagram
    actor User
    participant VideoSvc as Video Service
    participant Storage as Video Storage
    participant Transcoder as Transcode Worker

    User->>VideoSvc: Upload toàn bộ file video (1 lần, không chia chunk)
    VideoSvc->>Storage: Store raw file
    VideoSvc->>Transcoder: Transcode ra 1 độ phân giải duy nhất
    Transcoder-->>VideoSvc: Rendition ready
    VideoSvc->>VideoSvc: Publish video khi xử lý xong hoàn toàn
    VideoSvc-->>User: Video hiển thị công khai

    Note over User,VideoSvc: Nếu mất mạng giữa lúc upload, phải upload lại từ đầu
    Note over VideoSvc,Transcoder: Nếu transcode fail giữa chừng, phải chạy lại từ đầu
```
