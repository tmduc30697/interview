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
