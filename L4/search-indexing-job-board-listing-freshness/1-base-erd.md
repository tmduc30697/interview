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
