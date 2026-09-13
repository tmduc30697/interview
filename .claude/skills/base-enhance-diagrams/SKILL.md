---
name: base-enhance-diagrams
description: Đọc 1 đề bài mô tả 1 tính năng/thay đổi cần thêm vào hệ thống (path tới file, file đang mở, hoặc path tới folder mà đề bài là README.md bên trong), tách đề bài thành phần "base" (trạng thái hệ thống trước khi áp đề bài, tự suy luận từ ngữ cảnh) và phần "enhance" (chính nội dung đề bài), rồi vẽ ERD + sequence diagram bằng mermaid cho cả 2 giai đoạn để so sánh trực quan trước/sau khi triển khai. Dùng skill này khi người dùng muốn xem tổng quan dữ liệu và luồng nghiệp vụ trước và sau khi thêm 1 tính năng, hoặc muốn tài liệu hoá base + enhance bằng ERD/sequence diagram.
---

# Base vs Enhance diagram

Mục tiêu: từ 1 đề bài mô tả 1 thay đổi/tính năng cần thêm, dựng lại bức tranh dữ liệu (ERD) + luồng nghiệp vụ (sequence diagram) ở trạng thái **base** (trước khi áp đề bài) và trạng thái **sau khi có enhance** (sau khi áp đề bài), để người đọc so sánh trực quan tác động của thay đổi.

## Quy ước input đề bài

Người dùng cung cấp 1 trong 3 dạng sau:
- Path tới 1 file cụ thể — đó chính là đề bài.
- Không cung cấp path nào — lấy file đang mở trong editor làm đề bài.
- Path tới 1 folder — đề bài là file README.md nằm bên trong folder đó.

Toàn bộ file diagram tạo ra ở Bước 3-6 lưu **ngay trong folder chứa file đề bài** (folder đó nếu input là folder/README.md, hoặc folder cha của file nếu input là 1 file lẻ/file đang mở) — không tạo folder con, không lưu ra ngoài.

## Cách hiểu "base" và "enhance"

- **enhance** = toàn bộ nội dung đề bài. Đề bài luôn được hiểu là 1 yêu cầu thêm/sửa/nâng cấp lên 1 hệ thống, dù đề bài có nói tường minh kiểu "hiện tại đã có..." hay không.
- **base** = trạng thái hệ thống **ngay trước khi** áp đề bài này vào — phần này không có sẵn trong đề bài, phải tự suy luận từ ngữ cảnh (domain, actor, entity/dữ liệu được nhắc tới) xem hợp lý là hệ thống đã có gì trước đó để đề bài "có nghĩa".
- Base chỉ cần dựng phần **liên quan mật thiết tới enhance** — không suy diễn lan man ra 1 hệ thống hoàn chỉnh nếu đề bài không cần tới. Ví dụ đề bài "thêm nút Sign in with Google vào flow login hiện tại" → base chỉ cần entity/flow xoay quanh User + login bằng email/password, không tự bịa thêm module không liên quan (billing, notification...).
- Nếu đề bài mô tả 1 hệ thống hoàn toàn mới, không có gì tồn tại trước đó để suy luận (vd "xây dựng từ đầu 1 hệ thống X") — ghi rõ trong file base rằng base trống/hệ thống chưa tồn tại, và bỏ qua việc vẽ sequence diagram cho base vì không có flow nào để vẽ. Vẫn tạo file `1-base-erd.md` ghi nhận điều này để giữ mạch đánh số, rồi chuyển thẳng sang enhance.
- Nếu việc suy luận base có điểm mơ hồ **ảnh hưởng lớn** tới kết quả (vd base có thể hiểu là dùng SQL hoặc NoSQL, ảnh hưởng cấu trúc ERD) — hỏi lại người dùng. Nếu suy luận rõ ràng, hợp lý (đa số trường hợp) — tiến thẳng, không cần hỏi lại.

## Quy trình

### Bước 1 — Đọc đề bài

Xác định file đề bài đúng theo "Quy ước input đề bài" ở trên, đọc toàn bộ nội dung.

### Bước 2 — Suy luận base, xác định enhance

- Enhance = nguyên văn nội dung đề bài, không cần diễn giải lại.
- Suy luận base: liệt kê những entity/dữ liệu và action/flow nào **chắc chắn phải tồn tại trước đó** để đề bài này có nghĩa (vd đề bài nói "thêm trường phone_verified vào profile" → base chắc chắn đã có entity User/Profile với các trường cơ bản, và đã có flow "Đăng ký"/"Cập nhật profile"). Chỉ suy luận những gì thật sự phục vụ mục đích so sánh trước/sau, tránh bịa thêm chi tiết không liên quan.

### Bước 3 — [base] Vẽ ERD

Vẽ ERD (mermaid `erDiagram`) mô tả entity + quan hệ dữ liệu ở trạng thái base đã suy luận ở Bước 2. Lưu vào file `1-base-erd.md`.

Nếu base trống (hệ thống hoàn toàn mới — xem ghi chú ở trên), file này ghi rõ "chưa có entity nào tồn tại trước enhance" thay vì vẽ 1 ERD rỗng vô nghĩa hoặc bịa entity không có căn cứ.

### Bước 4 — [base] Liệt kê flow liên quan, vẽ sequence diagram

Liệt kê 1 số action/use case/flow chính ở base **liên quan mật thiết tới enhance** (là tiền đề cho enhance, hoặc sẽ bị enhance tác động trực tiếp) — không liệt kê flow không liên quan tới thay đổi sắp tới. Với mỗi flow, vẽ 1 sequence diagram (mermaid `sequenceDiagram`).

Đặt tên file: `2-base-<ten-flow>-sqd.md`, `3-base-<ten-flow>-sqd.md`, ... — số tăng dần tiếp theo ngay sau `1-base-erd.md`; `<ten-flow>` là tên flow viết tiếng Anh, dạng kebab-case, ngắn gọn (vd `login`, `update-profile`).

Nếu base trống, không có flow nào để vẽ — bỏ qua bước này, không tạo file nào ở đây; số thứ tự của enhance ERD ở Bước 5 sẽ là `2-enhance-erd.md` (tiếp ngay sau `1-base-erd.md`).

### Bước 5 — [enhance] Vẽ ERD

Vẽ ERD mô tả entity + quan hệ dữ liệu **sau khi** enhance được áp dụng lên base (entity/trường/quan hệ cũ từ base cộng với thay đổi mà enhance yêu cầu). Lưu vào file `N-enhance-erd.md`, với N là số tiếp theo ngay sau file base cuối cùng đã tạo ở Bước 3-4.

### Bước 6 — [enhance] Liệt kê flow chính, vẽ sequence diagram

Liệt kê action/use case/flow chính của hệ thống **sau khi** có enhance — gồm 2 loại:
- Flow đã liệt kê ở Bước 4 nhưng nay có thay đổi do enhance — giữ nguyên `<ten-flow>` để người đọc so sánh trực tiếp file base và file enhance cùng tên flow.
- Flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base.

Với mỗi flow, vẽ 1 sequence diagram. Đặt tên file: `N-enhance-<ten-flow>-sqd.md`, số tiếp tục tăng dần ngay sau file enhance ERD ở Bước 5.

### Bước 7 — Kỹ thuật vẽ & nội dung mỗi file

- ERD dùng mermaid `erDiagram`; sequence diagram dùng mermaid `sequenceDiagram` — cả 2 nhúng bằng code fence ```` ```mermaid ``` ````, render trực tiếp trong markdown preview của VSCode/GitHub.
- **Không dùng dấu `;` bên trong text của message/Note trong `sequenceDiagram`** — mermaid coi `;` là ký tự kết thúc statement nên sẽ gãy parser ngay sau dấu `;` đầu tiên (lỗi "Expecting ... got NEWLINE"). Dùng dấu `,` hoặc tách thành nhiều `Note` liên tiếp thay thế.
- Mỗi file là 1 tài liệu độc lập, bắt đầu bằng `# <tiêu đề>`, sau đó 1-2 câu mô tả: (1) đây là **base** hay **enhance**, (2) nếu là sequence diagram thì flow gì, (3) vì sao flow/entity này được chọn — liên quan gì tới enhance. Sau đó tới khối mermaid.
- File enhance của 1 flow đã có ở base nên nêu rõ trong câu mô tả "so với base, flow này thay đổi ở đâu" để làm nổi bật tác động của enhance.

### Bước 8 — Liệt kê lại toàn bộ file

Sau khi vẽ xong, liệt kê đường dẫn tất cả file đã tạo theo đúng thứ tự số, tóm tắt ngắn gọn base có gì / enhance thêm gì — không lặp lại nội dung diagram trong text vì file đã thể hiện đầy đủ.
