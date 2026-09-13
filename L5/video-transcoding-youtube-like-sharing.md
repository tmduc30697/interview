# Nền tảng chia sẻ video kiểu YouTube

**Hệ thống:** Một nền tảng cho phép user upload video và người khác xem lại (video-on-demand), quy mô hàng triệu video.

**Vai trò của flow:** Nhận file video gốc từ user, transcode ra nhiều độ phân giải/bitrate và đóng gói HLS/DASH để phát trên mọi thiết bị.

**Yêu cầu cụ thể:**
- Upload phải hỗ trợ file lớn (nhiều GB) bằng chunked/resumable upload, tiếp tục được nếu mất mạng giữa chừng.
- Sau khi upload xong, đưa vào hàng đợi transcode ra tối thiểu 3 độ phân giải (ví dụ 1080p/720p/480p) chạy song song, không block nhau nếu một rendition lỗi.
- Sinh thumbnail tự động (nhiều frame để user chọn) và file manifest HLS/DASH tương ứng.
- Nếu job transcode fail giữa chừng (worker crash, hết dung lượng tạm), phải retry lại đúng phần việc còn thiếu, không transcode lại từ đầu toàn bộ video.
- Video chỉ hiển thị "public" sau khi tối thiểu 1 rendition sẵn sàng; các rendition còn lại có thể hoàn thành dần (progressive availability).
- Có cơ chế dọn file gốc/file tạm sau X ngày để tối ưu chi phí storage, nhưng vẫn giữ được khả năng re-transcode nếu cần đổi codec sau này.

---

# ERD — Base (trước khi có pipeline transcode đa độ phân giải)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng chia sẻ video *trước khi* có pipeline upload resumable, multi-resolution, progressive availability. Đề bài giả định nền tảng đã cho user upload và xem lại video ở dạng đơn giản — nếu không có sẵn User/Video thì các yêu cầu về chunked upload, retry, cleanup sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ upload + phát video, không suy diễn thêm subscription, comment, recommendation...

```mermaid
erDiagram
    USER ||--o{ VIDEO : uploads

    USER {
        string user_id PK
        string display_name
        string status
    }

    VIDEO {
        string video_id PK
        string user_id FK
        string raw_storage_path
        string status
        datetime uploaded_at
        datetime published_at
    }
```

Ghi chú giả định: base là dạng upload nguyên khối (không resumable) và transcode 1 độ phân giải duy nhất, publish chỉ khi xử lý xong hoàn toàn — đây là điểm khởi đầu hợp lý nhất để so sánh với các yêu cầu nâng cấp toàn diện của đề bài.

---

<a id="2-base-upload-and-publish-video-sqd"></a>

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

---

# ERD — Enhance (sau khi có pipeline transcode đa độ phân giải)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 6 yêu cầu của đề bài — resumable upload theo chunk, transcode song song nhiều độ phân giải, sinh thumbnail + manifest HLS/DASH, retry đúng phần thiếu, progressive availability, và cleanup có khả năng re-transcode. So với base, `VIDEO` không còn gắn 1 file kết quả duy nhất mà tách thành nhiều `RENDITION` độc lập, publish được ngay khi có rendition đầu tiên thay vì chờ toàn bộ.

```mermaid
erDiagram
    USER ||--o{ VIDEO : uploads
    VIDEO ||--o{ UPLOAD_CHUNK : "uploaded via"
    VIDEO ||--o{ RENDITION : "transcoded into"
    VIDEO ||--o{ THUMBNAIL : "generated as"
    VIDEO ||--o| MANIFEST : "packaged as"
    VIDEO ||--o| RETENTION_POLICY : "governed by"

    USER {
        string user_id PK
        string display_name
        string status
    }

    VIDEO {
        string video_id PK
        string user_id FK
        string raw_storage_path
        string status
        boolean raw_deleted
        datetime uploaded_at
        datetime published_at
    }

    UPLOAD_CHUNK {
        string chunk_id PK
        string video_id FK
        int chunk_index
        string status
        int size_bytes
    }

    RENDITION {
        string rendition_id PK
        string video_id FK
        string resolution
        string status
        string storage_path
        string failed_segment_ref
    }

    THUMBNAIL {
        string thumbnail_id PK
        string video_id FK
        string storage_path
        int frame_position
        boolean is_selected
    }

    MANIFEST {
        string manifest_id PK
        string video_id FK
        string format
        string storage_path
    }

    RETENTION_POLICY {
        string policy_id PK
        string video_id FK
        int retain_days
        boolean original_cleaned_up
        boolean retranscode_possible
    }
```

---

# Sequence Diagram — Enhance: Upload and Publish Video

Đây là **enhance**, flow đã thay đổi so với base ([2-base-upload-and-publish-video-sqd.md](#2-base-upload-and-publish-video-sqd)): upload nay chia chunk (xem chi tiết resume ở [5-enhance-resumable-chunked-upload-sqd.md](#5-enhance-resumable-chunked-upload-sqd)), transcode nhiều độ phân giải song song không chặn nhau, và video publish ngay khi có rendition đầu tiên thay vì chờ toàn bộ như base.

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

---

<a id="5-enhance-resumable-chunked-upload-sqd"></a>

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

---

# Sequence Diagram — Enhance: Retry Failed Transcode

Đây là **enhance**, flow mới xử lý yêu cầu "nếu job transcode fail giữa chừng, phải retry đúng phần việc còn thiếu, không transcode lại từ đầu toàn bộ video".

```mermaid
sequenceDiagram
    participant Transcoder as Transcode Worker
    participant VideoSvc as Video Service
    participant Storage as Video Storage
    participant RetryQueue as Retry Queue

    Transcoder->>Storage: Transcode rendition theo từng segment
    Transcoder--xVideoSvc: Worker crash / hết dung lượng tạm giữa chừng

    VideoSvc->>Storage: Kiểm tra segment nào của RENDITION đã transcode xong
    Storage-->>VideoSvc: Danh sách segment done, failed_segment_ref cho phần còn thiếu

    VideoSvc->>RetryQueue: Enqueue retry job chỉ cho phần segment còn thiếu
    RetryQueue->>Transcoder: Dispatch worker mới, tiếp tục từ failed_segment_ref
    Transcoder->>Storage: Transcode nốt phần còn thiếu
    Transcoder-->>VideoSvc: Rendition hoàn chỉnh, không phải làm lại từ đầu
```

---

# Sequence Diagram — Enhance: Cleanup and Retranscode Original

Đây là **enhance**, flow mới xử lý yêu cầu dọn file gốc/file tạm sau X ngày để tối ưu chi phí lưu trữ, nhưng vẫn giữ khả năng re-transcode nếu cần đổi codec sau này.

```mermaid
sequenceDiagram
    participant Scheduler as Retention Scheduler
    participant VideoSvc as Video Service
    participant Storage as Video Storage
    actor Admin

    Scheduler->>VideoSvc: Check RETENTION_POLICY, video quá retain_days
    VideoSvc->>Storage: Xác nhận đã có đủ RENDITION cần thiết (đã publish ổn định)

    alt đủ điều kiện dọn
        VideoSvc->>Storage: Xóa raw file gốc, giữ lại các RENDITION đã transcode
        VideoSvc->>VideoSvc: Set original_cleaned_up = true
    end

    Note over VideoSvc,Storage: Sau này nếu cần đổi codec

    Admin->>VideoSvc: Yêu cầu re-transcode video (đổi codec mới)
    VideoSvc->>Storage: Kiểm tra retranscode_possible

    alt raw gốc đã xóa
        VideoSvc->>Storage: Dùng RENDITION chất lượng cao nhất còn lại làm nguồn để re-transcode
        Note over VideoSvc: Chất lượng re-transcode giới hạn bởi rendition cao nhất còn giữ lại
    else raw gốc còn giữ
        VideoSvc->>Storage: Re-transcode trực tiếp từ raw file gốc
    end
    VideoSvc-->>Admin: Re-transcode hoàn tất
```
