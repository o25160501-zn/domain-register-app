# Section 3 — User Stories

---

## 3.1 Personas

### Persona 1 — Tuấn (Solo Developer)
- **Vai trò:** Freelancer, tự build side project
- **Kỹ năng:** Biết dùng API, quen Cloudflare, không muốn mất thời gian làm thủ công
- **Mục tiêu:** Đăng ký 1–3 domain miễn phí cho các project demo, staging environment
- **Pain point:** Phải vào 2 dashboard khác nhau, copy-paste nameserver thủ công, quên đã tạo domain nào

### Persona 2 — Linh (DevOps tại startup nhỏ)
- **Vai trò:** Quản lý infra cho team 5 người
- **Kỹ năng:** Thành thạo CLI và API automation
- **Mục tiêu:** Quản lý nhiều domain cho các môi trường dev/staging, theo dõi trạng thái từng domain
- **Pain point:** Không có audit trail, không biết ai tạo domain nào và lúc nào

### Persona 3 — Minh (Học sinh / Sinh viên)
- **Vai trò:** Học lập trình, build portfolio
- **Kỹ năng:** Biết cơ bản về web, mới làm quen DNS
- **Mục tiêu:** Có một domain miễn phí để deploy project cá nhân
- **Pain point:** Quy trình DNS phức tạp, dễ nhầm bước, không biết nameserver là gì

---

## 3.2 User Stories

---

### Epic 1 — Thiết lập API Credentials

**US-001 — Lưu DigitalPlat Token**
> As a **Tuấn (developer)**,
> I want to **nhập và lưu DigitalPlat API Token của tôi một lần**,
> So that **tôi không cần paste token lại mỗi lần đăng ký domain mới**.

**Acceptance Criteria:**
- [ ] Có form nhập DigitalPlat Bearer Token với label rõ ràng
- [ ] Sau khi lưu, token hiển thị dạng masked (e.g. `eyJhbGc...****...xYz`)
- [ ] Có nút "Test Connection" gọi thử API để xác nhận token hợp lệ
- [ ] Toast notification hiển thị "Token saved successfully" hoặc lỗi cụ thể
- [ ] Token được ghi vào Firebase node `/settings/credentials/dpdns_token`

---

**US-002 — Lưu Cloudflare Global API Key**
> As a **Tuấn (developer)**,
> I want to **lưu Cloudflare Email và Global API Key của tôi**,
> So that **ứng dụng có thể tự động tạo zone Cloudflare thay tôi**.

**Acceptance Criteria:**
- [ ] Form gồm 2 trường: Email và Global API Key
- [ ] Validate email format trước khi lưu
- [ ] Gọi `GET https://api.cloudflare.com/client/v4/user` để xác nhận credentials hợp lệ
- [ ] Hiển thị tên tài khoản Cloudflare sau khi xác nhận thành công
- [ ] Credentials lưu vào Firebase `/settings/credentials/cloudflare`

---

### Epic 2 — Đăng ký Domain

**US-010 — Đăng ký domain mới**
> As a **Tuấn (developer)**,
> I want to **nhập tên domain và bấm một nút để hoàn thành toàn bộ quy trình đăng ký**,
> So that **tôi không phải vào DigitalPlat và Cloudflare riêng lẻ nữa**.

**Acceptance Criteria:**
- [ ] Form có trường nhập subdomain và dropdown chọn namespace (`.dpdns.org`, `.us.kg`, `.qzz.io`, `.xx.kg`)
- [ ] Validate format domain: chỉ chứa `[a-z0-9-]`, không bắt đầu/kết thúc bằng `-`
- [ ] Hiển thị step indicator với 3 bước: ① Tạo Cloudflare Zone → ② Lấy Nameserver → ③ Đăng ký DPDNS
- [ ] Mỗi bước hiển thị trạng thái: ⏳ Processing → ✅ Done hoặc ❌ Failed
- [ ] Sau khi hoàn thành, domain xuất hiện ngay trong danh sách (realtime)
- [ ] Nếu thất bại ở bước 3, Cloudflare zone đã tạo ở bước 1 được xoá (rollback)

---

**US-011 — Xem tiến trình đăng ký**
> As a **Minh (sinh viên)**,
> I want to **thấy rõ từng bước đang diễn ra trong quá trình đăng ký**,
> So that **tôi hiểu hệ thống đang làm gì và biết khi nào xong**.

**Acceptance Criteria:**
- [ ] Hiển thị 3 bước với icon trạng thái
- [ ] Mỗi bước có mô tả ngắn bằng tiếng Việt (e.g. "Đang tạo zone trên Cloudflare...")
- [ ] Thời gian thực hiện từng bước được hiển thị sau khi hoàn thành
- [ ] Sau khi hoàn thành toàn bộ: hiển thị summary card với nameservers

---

### Epic 3 — Xem & Quản lý Domain

**US-020 — Xem danh sách domain**
> As a **Linh (DevOps)**,
> I want to **thấy danh sách tất cả domain đã đăng ký với đầy đủ thông tin**,
> So that **tôi biết domain nào đang hoạt động và được tạo lúc nào**.

**Acceptance Criteria:**
- [ ] Danh sách hiển thị: tên domain, namespace, nameservers, created_at, updated_at
- [ ] Dữ liệu cập nhật realtime (Firebase `onValue` listener)
- [ ] Hiển thị "Không có domain nào" khi list rỗng, kèm CTA "Đăng ký domain đầu tiên"
- [ ] Có thể copy nameserver vào clipboard bằng một click

---

**US-030 — Sửa nameserver domain**
> As a **Linh (DevOps)**,
> I want to **cập nhật nameserver của domain đã đăng ký**,
> So that **tôi có thể chuyển domain sang DNS provider khác nếu cần**.

**Acceptance Criteria:**
- [ ] Có nút Edit trên mỗi domain row
- [ ] Form edit hiển thị nameservers hiện tại, cho phép sửa
- [ ] Sau khi save, hệ thống gọi DigitalPlat API để cập nhật NS
- [ ] `updated_at` được cập nhật trong Firebase
- [ ] Hiển thị trạng thái "Updating..." trong quá trình gọi API

---

**US-040 — Xoá domain khỏi danh sách**
> As a **Tuấn (developer)**,
> I want to **xoá domain không còn dùng nữa khỏi danh sách quản lý**,
> So that **danh sách luôn gọn gàng và chỉ hiện những domain đang active**.

**Acceptance Criteria:**
- [ ] Có nút Delete trên mỗi domain row
- [ ] Confirmation dialog hiển thị tên domain và cảnh báo hành động không thể hoàn tác
- [ ] Checkbox opt-in: "Đồng thời xoá Cloudflare Zone" (mặc định unchecked)
- [ ] Sau khi xác nhận, record được xoá khỏi Firebase ngay lập tức
- [ ] Toast notification: "Domain `myapp.dpdns.org` đã được xoá"

---

## 3.3 Story Map (Priority Matrix)

```
HIGH VALUE / LOW EFFORT (Do First)
  US-001  Lưu DigitalPlat Token
  US-002  Lưu Cloudflare Credentials
  US-010  Đăng ký domain mới (core flow)
  US-020  Xem danh sách domain

HIGH VALUE / HIGH EFFORT (Plan Carefully)
  US-011  Step indicator đăng ký
  US-030  Sửa nameserver

MEDIUM VALUE / LOW EFFORT (Quick Wins)
  US-040  Xoá domain

LOW VALUE / HIGH EFFORT (Backlog)
  US-033  Thêm ghi chú domain
  US-043  Xoá Cloudflare zone khi delete
```
