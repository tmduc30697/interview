# Sequence - Enhance: Compliance chạy audit query trên read replica

Đây là **enhance**, flow hoàn toàn mới phát sinh từ đề bài, chưa tồn tại ở base. Bộ phận compliance cần chạy các query full-scan với tổ hợp cột thay đổi liên tục theo từng đợt kiểm toán, không thể index hết - flow này tách các truy vấn đó sang read replica riêng để không cạnh tranh index/buffer pool với luồng giao dịch thời gian thực (OLTP) trên primary.

```mermaid
sequenceDiagram
    actor Compliance as Compliance Analyst
    participant Tool as Audit/Reporting Tool
    participant Replica as Postgres Read Replica
    participant Primary as Postgres Primary - transactions

    Primary-->>Replica: Streaming replication liên tục (dữ liệu transactions)

    Compliance->>Tool: Chọn tiêu chí audit đợt này (tổ hợp cột tuỳ đợt)
    Tool->>Replica: SELECT ... FROM transactions WHERE <tổ hợp cột không cố định>

    Note over Replica: Khong the index het moi to hop cot co the doi bat ky luc nao
    Note over Replica: Chap nhan Seq Scan / full-scan tren replica

    Replica-->>Tool: Kết quả audit (có thể có độ trễ replication)
    Tool-->>Compliance: Báo cáo audit

    Note over Primary: Trong khi do, Primary van phuc vu insert/update<br/>giao dich thoi gian thuc binh thuong
    Note over Primary: Khong bi anh huong boi tai I/O nang<br/>tu cac query audit full-scan
```

