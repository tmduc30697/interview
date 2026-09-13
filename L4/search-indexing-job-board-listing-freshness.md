# Nền tảng tuyển dụng với yêu cầu độ tươi mới của tin đăng

**Hệ thống:** Một nền tảng tuyển dụng cho phép ứng viên tìm việc theo kỹ năng/địa điểm/mức lương, nhà tuyển dụng đăng tin và có thể đóng tin bất cứ lúc nào khi đã tuyển đủ.

**Vai trò của flow:** Lập chỉ mục tin tuyển dụng với các thuộc tính lọc (kỹ năng, địa điểm, mức lương), phục vụ tìm kiếm và đảm bảo tin hết hạn/đã tuyển đủ biến mất khỏi kết quả ngay lập tức.

**Yêu cầu cụ thể:**
- Khi nhà tuyển dụng đánh dấu tin đã tuyển đủ hoặc chủ động đóng tin, tin đó phải biến mất khỏi kết quả tìm kiếm ngay lập tức, không chờ tới chu kỳ đồng bộ định kỳ — vì ứng viên nộp hồ sơ vào tin đã đóng gây trải nghiệm xấu và tổn hại uy tín nền tảng nhiều hơn so với các loại nội dung khác chấp nhận độ trễ index cao hơn.
- Tin tuyển dụng có ngày hết hạn tự động phải được gỡ khỏi index đúng thời điểm hết hạn mà không cần nhà tuyển dụng chủ động thao tác, xử lý đúng múi giờ và không để lệch vài giờ khiến tin hết hạn vẫn hiển thị hoặc tin còn hạn bị gỡ nhầm.
- Tìm kiếm theo mức lương phải xử lý được các tin để khoảng lương (min-max) thay vì một con số cố định, và một số tin không công khai mức lương — bộ lọc phải trả kết quả đúng logic giao khoảng khi ứng viên lọc theo một mức lương mong muốn cụ thể, không loại nhầm các tin có khoảng lương chỉ giao một phần với tiêu chí lọc.
- Kỹ năng yêu cầu trong tin đăng thường được nhà tuyển dụng gõ tự do, trong khi ứng viên tìm theo tên kỹ năng chuẩn — autocomplete và index phải ánh xạ được các cách viết/tên gọi khác nhau của cùng một kỹ năng (viết tắt, đồng nghĩa) về cùng một khái niệm để không bỏ lỡ tin phù hợp chỉ vì khác cách gõ.
- Khi một công ty đăng số lượng lớn tin cùng lúc hoặc gỡ hàng loạt tin cùng lúc, pipeline index phải xử lý được đợt thay đổi hàng loạt này mà không tạo ra độ trễ lan sang việc cập nhật index cho các tin đăng bình thường khác của nhà tuyển dụng khác đang diễn ra song song.

---

# ERD — Base (trước khi cải thiện độ tươi mới)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng tuyển dụng *trước khi* cải thiện độ tươi mới của index. Đề bài mô tả nền tảng "đã cho phép" tìm việc theo kỹ năng/địa điểm/mức lương và nhà tuyển dụng đã đăng/đóng tin được — nghĩa là index tìm kiếm đã tồn tại, chỉ là đồng bộ theo chu kỳ định kỳ (batch), chưa tức thời. Đây là điểm base cần có để yêu cầu "tin đóng phải biến mất ngay lập tức" có nghĩa.

```mermaid
erDiagram
    EMPLOYER ||--o{ JOB_POSTING : posts
    JOB_POSTING ||--o| JOB_INDEX_DOCUMENT : "synced periodically (batch)"

    EMPLOYER {
        string employer_id PK
        string company_name
    }

    JOB_POSTING {
        string job_id PK
        string employer_id FK
        string title
        string skills_text
        string location
        decimal salary_min
        decimal salary_max
        boolean salary_visible
        string status
        date expires_at
        datetime updated_at
    }

    JOB_INDEX_DOCUMENT {
        string job_id PK
        string title
        string skills_text
        string location
        decimal salary_min
        decimal salary_max
        string status
        datetime last_synced_at
    }
```

---

# Sequence Diagram — Base: Post Job

Đây là **base**, flow "đăng tin tuyển dụng" — cho thấy tin mới chỉ vào index sau chu kỳ đồng bộ định kỳ (batch), là tiền đề cho vấn đề độ trễ mà enhance phải giải quyết ở cả chiều đăng lẫn chiều đóng tin.

```mermaid
sequenceDiagram
    actor Employer
    participant JobSvc as Job Posting Service
    participant DB as Job Postings DB
    participant BatchSync as Periodic Sync Job (batch)
    participant Index as Job Search Index

    Employer->>JobSvc: Đăng tin tuyển dụng mới
    JobSvc->>DB: INSERT INTO job_postings (status = active)
    DB-->>JobSvc: job_id created
    JobSvc-->>Employer: Tin đã đăng

    Note over DB,BatchSync: Tin chưa xuất hiện trong kết quả tìm kiếm ngay
    BatchSync->>DB: Chạy theo chu kỳ định kỳ, quét các thay đổi
    BatchSync->>Index: Đồng bộ tin mới vào index
    Index-->>BatchSync: Indexed

    Note over BatchSync,Index: Có độ trễ giữa lúc đăng tin và lúc tin xuất hiện trong tìm kiếm, phụ thuộc chu kỳ batch
```

---

<a id="3-base-close-job-sqd"></a>

# Sequence Diagram — Base: Close Job

Đây là **base**, flow "đóng tin đã tuyển đủ" — vấn đề cốt lõi mà enhance phải giải quyết: tin bị đóng trong DB nhưng vẫn hiển thị trong kết quả tìm kiếm cho tới chu kỳ đồng bộ định kỳ kế tiếp, khiến ứng viên vẫn nộp hồ sơ vào tin đã đóng.

```mermaid
sequenceDiagram
    actor Employer
    participant JobSvc as Job Posting Service
    participant DB as Job Postings DB
    participant BatchSync as Periodic Sync Job (batch)
    participant Index as Job Search Index
    actor Candidate

    Employer->>JobSvc: Đánh dấu tin đã tuyển đủ / đóng tin
    JobSvc->>DB: UPDATE job_postings SET status = closed
    DB-->>JobSvc: Updated
    JobSvc-->>Employer: Tin đã đóng

    Note over Index: Index chưa được cập nhật, tin vẫn còn status = active trong index
    Candidate->>Index: Tìm kiếm việc làm
    Index-->>Candidate: Tin đã đóng vẫn xuất hiện trong kết quả (chưa tới chu kỳ sync)

    BatchSync->>DB: Chạy theo chu kỳ định kỳ
    BatchSync->>Index: Đồng bộ status = closed
    Index-->>BatchSync: Indexed, tin biến mất khỏi kết quả từ lúc này
```

---

# ERD — Enhance (sau khi cải thiện độ tươi mới)

Đây là **enhance**: ERD base cộng với `SKILL_SYNONYM_MAP` để ánh xạ các cách gõ kỹ năng khác nhau về cùng khái niệm, và `JOB_INDEX_DOCUMENT` được bổ sung cơ chế đồng bộ sự kiện tức thời thay vì batch định kỳ. So với base, việc đồng bộ index không còn qua `Periodic Sync Job` mà qua `INDEX_SYNC_EVENT` được xử lý gần như ngay lập tức, và `JOB_INDEX_DOCUMENT` có thêm `skills_normalized` để phục vụ autocomplete theo tên kỹ năng chuẩn.

```mermaid
erDiagram
    EMPLOYER ||--o{ JOB_POSTING : posts
    JOB_POSTING ||--o{ INDEX_SYNC_EVENT : "triggers on change"
    INDEX_SYNC_EVENT ||--o| JOB_INDEX_DOCUMENT : updates
    SKILL_SYNONYM_MAP ||--o{ JOB_INDEX_DOCUMENT : "normalizes skills for"

    EMPLOYER {
        string employer_id PK
        string company_name
    }

    JOB_POSTING {
        string job_id PK
        string employer_id FK
        string title
        string skills_text
        string location
        decimal salary_min
        decimal salary_max
        boolean salary_visible
        string status
        date expires_at
        string timezone
        datetime updated_at
    }

    JOB_INDEX_DOCUMENT {
        string job_id PK
        string title
        string skills_text
        string skills_normalized
        string location
        decimal salary_min
        decimal salary_max
        boolean salary_visible
        string status
        datetime expires_at
        datetime last_synced_at
    }

    SKILL_SYNONYM_MAP {
        string mapping_id PK
        string raw_term
        string canonical_skill_id
    }

    INDEX_SYNC_EVENT {
        string event_id PK
        string job_id FK
        string change_type
        string status
        datetime created_at
        datetime processed_at
    }
```

---

# Sequence Diagram — Enhance: Close Job

Đây là **enhance**, flow "đóng tin" đã tồn tại ở base ([3-base-close-job-sqd.md](#3-base-close-job-sqd)) nay thay đổi cốt lõi: thay vì chờ chu kỳ batch, việc đóng tin phát ngay `INDEX_SYNC_EVENT` và index được cập nhật tức thời, để tin biến mất khỏi kết quả tìm kiếm ngay lập tức.

```mermaid
sequenceDiagram
    actor Employer
    participant JobSvc as Job Posting Service
    participant DB as Job Postings DB
    participant Queue as Index Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Job Search Index
    actor Candidate

    Employer->>JobSvc: Đánh dấu tin đã tuyển đủ / đóng tin
    JobSvc->>DB: UPDATE job_postings SET status = closed
    DB-->>JobSvc: Updated
    JobSvc->>Queue: Publish INDEX_SYNC_EVENT (job_id, change_type = closed), priority cao

    Queue->>Indexer: Deliver event ngay lập tức
    Indexer->>Index: UPDATE job_index_document SET status = closed
    Index-->>Indexer: Indexed

    JobSvc-->>Employer: Tin đã đóng
    Candidate->>Index: Tìm kiếm việc làm
    Index-->>Candidate: Tin đã đóng không còn xuất hiện, gần như ngay sau khi đóng
```

---

# Sequence Diagram — Enhance: Job Expiry Sweep

Đây là **enhance**, flow hoàn toàn mới: tin có ngày hết hạn phải tự động bị gỡ khỏi index đúng thời điểm hết hạn, xử lý đúng múi giờ, mà không cần nhà tuyển dụng thao tác.

```mermaid
sequenceDiagram
    participant Scheduler as Expiry Sweep Scheduler
    participant DB as Job Postings DB
    participant Queue as Index Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Job Search Index

    loop Chạy liên tục theo mốc thời gian nhỏ, ví dụ mỗi phút
        Scheduler->>DB: Tìm job_posting có expires_at (quy đổi đúng timezone) đã tới hoặc vừa qua, status vẫn active
        DB-->>Scheduler: Danh sách tin vừa hết hạn

        loop for each job vừa hết hạn
            Scheduler->>DB: UPDATE job_postings SET status = expired
            Scheduler->>Queue: Publish INDEX_SYNC_EVENT (job_id, change_type = expired)
        end
    end

    Queue->>Indexer: Deliver expiry events
    Indexer->>Index: UPDATE job_index_document SET status = expired
    Index-->>Indexer: Indexed

    Note over Scheduler,Index: Mốc quét đủ nhỏ và tính đúng timezone để tin hết hạn không hiển thị quá lâu, và tin còn hạn không bị gỡ nhầm
```

---

# Sequence Diagram — Enhance: Search Jobs

Đây là **enhance**, flow tìm kiếm việc làm hoàn toàn mới so với base (base chưa mô tả chi tiết cách lọc): xử lý lọc theo khoảng lương (giao khoảng, kể cả tin không công khai lương), và autocomplete kỹ năng ánh xạ qua `SKILL_SYNONYM_MAP` để không bỏ lỡ tin dùng cách gõ khác.

```mermaid
sequenceDiagram
    actor Candidate
    participant WebApp as Job Board App
    participant Search as Search Service
    participant SynonymMap as Skill Synonym Map
    participant Index as Job Search Index

    Candidate->>WebApp: Gõ kỹ năng (viết tắt/đồng nghĩa) và mức lương mong muốn
    WebApp->>Search: Search request (skill_term, desired_salary, location)

    Search->>SynonymMap: Tra canonical_skill_id cho skill_term đã gõ
    SynonymMap-->>Search: Danh sách tên gọi khác nhau cùng ánh xạ 1 khái niệm

    Search->>Index: Query theo skills_normalized, filter salary_min <= desired_salary <= salary_max
    Index-->>Search: Danh sách tin khớp, bao gồm cả tin có khoảng lương giao 1 phần với mức mong muốn

    Search->>Search: Loại các tin không công khai lương khỏi filter lương nhưng vẫn giữ nếu khớp kỹ năng/địa điểm
    Search-->>WebApp: Kết quả tìm kiếm
    WebApp-->>Candidate: Hiển thị tin phù hợp
```

---

# Sequence Diagram — Enhance: Bulk Post Jobs

Đây là **enhance**, flow hoàn toàn mới: khi 1 công ty đăng hoặc gỡ hàng loạt tin cùng lúc, pipeline index phải xử lý đợt thay đổi lớn này mà không làm chậm việc cập nhật index cho các tin đăng bình thường của nhà tuyển dụng khác đang diễn ra song song.

```mermaid
sequenceDiagram
    actor Employer as Nhà tuyển dụng (đăng hàng loạt)
    actor OtherEmployer as Nhà tuyển dụng khác (đăng bình thường)
    participant JobSvc as Job Posting Service
    participant Queue as Index Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Job Search Index

    par Đăng hàng loạt
        Employer->>JobSvc: Đăng/gỡ hàng trăm tin cùng lúc
        JobSvc->>Queue: Publish nhiều INDEX_SYNC_EVENT, gắn lane = bulk
    and Đăng bình thường song song
        OtherEmployer->>JobSvc: Đăng 1 tin bình thường
        JobSvc->>Queue: Publish INDEX_SYNC_EVENT, lane = realtime
    end

    Queue->>Indexer: Route event theo lane riêng biệt, bulk không chặn realtime
    Indexer->>Index: Xử lý lane realtime ngay lập tức
    Index-->>Indexer: Tin của nhà tuyển dụng khác lên index ngay, không bị ảnh hưởng

    loop Xử lý lane bulk theo batch, throttled
        Indexer->>Index: Cập nhật index cho từng lô tin hàng loạt
        Index-->>Indexer: Batch indexed
    end

    Note over Queue,Indexer: Tách lane đảm bảo đợt đăng/gỡ hàng loạt không làm trễ index của các nhà tuyển dụng khác
```
