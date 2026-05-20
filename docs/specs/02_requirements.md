# Section 2 — Requirements

---

## 2.1 Functional Requirements

### Module A — API Credentials Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-001 | Người dùng có thể nhập và lưu **DigitalPlat API Token** (Bearer) vào hệ thống. | MUST |
| FR-002 | Người dùng có thể nhập và lưu **Cloudflare Email** và **Global API Key** vào hệ thống. | MUST |
| FR-003 | API credentials được lưu vào Firebase Realtime Database dưới node `/settings/credentials`. | MUST |
| FR-004 | Credentials được hiển thị dạng masked (ẩn ký tự giữa), có nút "Reveal" để xem toàn bộ. | MUST |
| FR-005 | Người dùng có thể cập nhật (overwrite) credentials bất kỳ lúc nào. | MUST |
| FR-006 | Hệ thống kiểm tra tính hợp lệ của credentials bằng cách gọi thử API trước khi lưu (validation call). | SHOULD |

---

### Module B — Domain Registration

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-010 | Người dùng nhập subdomain muốn đăng ký và chọn namespace (`.dpdns.org`, `.us.kg`, `.qzz.io`, `.xx.kg`). | MUST |
| FR-011 | Hệ thống kiểm tra domain có khả dụng (available) trước khi tiến hành đăng ký. | SHOULD |
| FR-012 | Hệ thống tự động gọi **Cloudflare API `POST /zones`** để tạo zone mới với domain đó. | MUST |
| FR-013 | Sau khi tạo zone thành công, hệ thống trích xuất `name_servers[]` từ response Cloudflare. | MUST |
| FR-014 | Hệ thống tự động gọi **DigitalPlat API** để đăng ký domain với nameservers vừa lấy từ Cloudflare. | MUST |
| FR-015 | Toàn bộ quá trình (bước FR-012 → FR-014) hiển thị tiến trình từng bước (step indicator) cho người dùng. | MUST |
| FR-016 | Nếu bất kỳ bước nào thất bại, hệ thống hiển thị lỗi rõ ràng và rollback (xoá zone Cloudflare nếu DPDNS thất bại). | MUST |

---

### Module C — Domain Listing & Detail

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-020 | Danh sách tất cả domain đã đăng ký được hiển thị realtime từ Firebase. | MUST |
| FR-021 | Mỗi domain entry hiển thị: tên domain, nameservers, Cloudflare zone_id, trạng thái, `created_at`, `updated_at`. | MUST |
| FR-022 | Danh sách domain được sắp xếp theo `created_at` mới nhất lên đầu (mặc định). | SHOULD |
| FR-023 | Người dùng có thể xem chi tiết đầy đủ của một domain bằng cách click vào. | SHOULD |

---

### Module D — Domain Edit

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-030 | Người dùng có thể chỉnh sửa nameservers của domain đã đăng ký. | MUST |
| FR-031 | Khi nameserver được cập nhật, hệ thống gọi DigitalPlat API để cập nhật NS đồng thời. | MUST |
| FR-032 | Sau khi cập nhật thành công, `updated_at` trong Firebase được cập nhật. | MUST |
| FR-033 | Người dùng có thể thêm ghi chú (notes/description) cho từng domain. | COULD |

---

### Module E — Domain Delete

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-040 | Người dùng có thể xoá domain khỏi danh sách quản lý của ứng dụng. | MUST |
| FR-041 | Trước khi xoá, hệ thống hiển thị confirmation dialog với tên domain để xác nhận. | MUST |
| FR-042 | Xoá domain chỉ xoá record khỏi Firebase; **không** tự động xoá zone Cloudflare hay domain DigitalPlat (để tránh mất domain do nhầm). | MUST |
| FR-043 | Hệ thống cung cấp tuỳ chọn đồng thời xoá Cloudflare zone khi xoá domain (checkbox opt-in). | COULD |

---

## 2.2 Non-Functional Requirements

### Performance

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-001 | Thời gian hoàn thành luồng đăng ký domain (Cloudflare + DPDNS) kể từ khi bấm "Register". | ≤ 15 giây |
| NFR-002 | Firebase Realtime Database cập nhật danh sách domain sau khi tạo/sửa/xoá. | ≤ 2 giây |
| NFR-003 | Thời gian tải trang đầu tiên (First Contentful Paint). | ≤ 3 giây |

### Security

| ID | Requirement | Ghi chú |
|----|-------------|---------|
| NFR-010 | API credentials (Token, API Key) không được lưu dưới dạng plaintext trong client-side storage (localStorage, sessionStorage). | Xem Section 7 |
| NFR-011 | Firebase Database Rules phải yêu cầu xác thực (authenticated) trước khi đọc/ghi. | Firebase Auth required |
| NFR-012 | Mọi giao tiếp với API (DPDNS, Cloudflare, Firebase) phải qua HTTPS. | Enforced by default |
| NFR-013 | API credentials trong Firebase được lưu dưới dạng mã hoá hoặc dùng Firebase Secret Manager. | Xem Section 7 |

### Reliability

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-020 | Hệ thống xử lý lỗi API timeout (DPDNS hoặc Cloudflare không phản hồi) và hiển thị thông báo lỗi thay vì crash. | 100% lỗi được bắt |
| NFR-021 | Retry logic: nếu Cloudflare API thất bại lần đầu do network, tự động retry tối đa 2 lần. | 2 retries |
| NFR-022 | Firebase offline persistence được bật để ứng dụng vẫn hiển thị dữ liệu khi mất kết nối tạm thời. | Enabled |

### Usability

| ID | Requirement | Ghi chú |
|----|-------------|---------|
| NFR-030 | Giao diện responsive, hoạt động trên desktop và mobile (≥ 375px width). | Mobile-friendly |
| NFR-031 | Mọi action có thể gây mất dữ liệu (delete) phải có bước xác nhận rõ ràng. | UX safety |
| NFR-032 | Trạng thái loading/processing phải hiển thị rõ ràng trong quá trình gọi API. | Spinner / progress |

### Maintainability

| ID | Requirement | Ghi chú |
|----|-------------|---------|
| NFR-040 | Code tách biệt rõ ràng giữa: UI layer, API service layer, Firebase service layer. | Clean architecture |
| NFR-041 | API endpoints và base URLs được cấu hình qua environment variables (`.env`), không hardcode. | 12-factor app |
| NFR-042 | Tất cả API calls được log (request/response summary) vào browser console ở development mode. | Debug friendly |
