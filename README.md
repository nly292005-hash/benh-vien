# PHÒNG KHÁM NHI HÀ NỘI — BÁC SĨ HANH

**Hệ thống đặt lịch khám trực tuyến – Mã nguồn MERN (React + Express + MongoDB/Mongoose).**

> **Bản cập nhật theo barem WEB:** xem `BAREM_FIX_REPORT.md` và bảng `DOI_CHIEU_BAREM_WEB.xlsx` trong dự án. Bổ sung React lazy/HOC/React.memo/useMemo/Formik/Yup/ErrorBoundary, phân trang UI/API, validation ngày giờ, refresh token rotation và tài liệu OpenAPI (`docs/API_OPENAPI.yaml`).

Dự án được phát triển từ báo cáo bài tập lớn **“QLDA-nhóm-16.docx”**, Học viện Ngân hàng, môn Quản lý dự án Công nghệ thông tin. Đây là **bản mã nguồn MVP học tập**, không phải hệ thống khám chữa bệnh được nghiệm thu triển khai thực tế. Chỉ sử dụng **dữ liệu giả lập**. Không nhập thông tin trẻ em/bệnh nhân thực tế khi chưa có đánh giá bảo mật, pháp lý và vận hành.

## 1. Cài đặt

Yêu cầu: **Node.js 20+**, npm và **MongoDB** (máy local hoặc MongoDB Atlas). Nên dùng **Node 22 LTS**. MongoDB cần chạy trước khi khởi động server.

```bash
# Mở terminal tại thư mục clinic-booking
npm install
npm --prefix server install
npm --prefix client install

# Tạo 2 file môi trường bằng cách copy mẫu:
# server/.env.example -> server/.env
# client/.env.example -> client/.env
```

Chỉnh `server/.env`:

- `MONGODB_URI`: ví dụ `mongodb://127.0.0.1:27017/phong-kham-nhi`.
- `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET`: hai chuỗi **khác nhau** dài ít nhất 32 ký tự.
- `MEDICAL_ENCRYPTION_KEY`: khóa AES-256, **64 ký tự hexadecimal**. Mất khóa = không đọc lại được dữ liệu y tế đã mã hóa.
- `ADMIN_EMAIL`, `ADMIN_PASSWORD`: email và mật khẩu admin dùng cho lệnh `seed` (mật khẩu tối thiểu 12 ký tự, không để mặc định).
- `BACKUP_ENCRYPTION_KEY`: khóa AES-256 thứ hai, riêng biệt, dài 64 ký tự hexadecimal. Không để chung với source code.
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`: cấu hình gửi email qua SMTP; nếu bỏ trống, thông báo **chỉ hiển thị trong app**, còn email / đặt lại mật khẩu qua email không hoạt động.

**Tạo khóa bí mật** (chạy từng lệnh trong terminal):

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"  # JWT secret
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"  # AES key
```

Tạo lần lượt 2 JWT secret và 2 AES key **riêng**, lưu ở nơi an toàn. **Tuyệt đối không commit `.env` lên GitHub**.

Sau khi cấu hình:

```bash
npm --prefix server run seed   # Khởi tạo 3 dịch vụ & admin, không ghi đè tài khoản có sẵn
npm run dev                    # Chạy đồng thời server và frontend
```

Truy cập:

- Frontend: http://localhost:5173
- Backend API: http://localhost:8080/api/health

Đăng nhập Admin bằng cặp `ADMIN_EMAIL` / `ADMIN_PASSWORD` đã cấu hình trong `.env`. Admin tạo tài khoản **DOCTOR**, **RECEPTIONIST**, **MANAGER** tại **Quản trị hệ thống**. Người dùng **PARENT** tự đăng ký ở `/register`.

**Luồng demo tối thiểu**:

1. Admin tạo bác sĩ, lễ tân và quản lý; bác sĩ cần có chuyên khoa.
2. Quản lý đăng nhập, tạo lịch trực tương lai cho bác sĩ qua trang **Ca và yêu cầu đổi ca**.
3. Phụ huynh đăng ký → tạo hồ sơ của bé kèm đồng ý xử lý dữ liệu → đặt lịch → xem lịch.
4. Lễ tân đăng nhập, tìm lịch theo ngày → xác nhận → check-in.
5. Bác sĩ đăng nhập → chọn bệnh nhi → cảnh báo dị ứng → bắt đầu khám → lưu chẩn đoán, đơn thuốc, lời dặn.
6. Phụ huynh xem **Hồ sơ bé > Lịch sử**, chat với bác sĩ và đánh giá sau khám.
7. Quản lý xem thống kê, đổi giá dịch vụ hoặc xử lý yêu cầu thay đổi ca.

## 2. Cấu trúc

```text
clinic-booking/
  client/                      # React / Vite / React Router / Axios
    src/
      components/              # Navigation, common UI
      context/                 # Auth Context
      pages/                   # Screen cho 5 vai trò
      api.js                   # API client + refresh token
      styles.css               # UI responsive tiếng Việt
  server/
    src/
      models/                  # Mongoose collections + unique index
      routes/                  # REST API theo 5 nhóm nghiệp vụ
      middleware/              # JWT authentication + RBAC
      lib/                     # AES encryption, slots, validation, mail
      services/                # Slot availability, data access, backups
      jobs/                    # Nhắc lịch và backup theo lịch
      index.js                 # Cấu hình Express, lỗi, CORS
      seed.js                  # Tạo admin và 3 dịch vụ
```

## 3. API chính

Tất cả endpoint trừ `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh`, `/api/auth/forgot-password`, `/api/auth/reset-password`, `/api/health` đều yêu cầu `Authorization: Bearer <ACCESS_TOKEN>`.

| Nhóm | Endpoint (tóm tắt) | Quyền |
|---|---|---|
| Auth | `POST /api/auth/register`, `login`, `refresh`, `logout`, `forgot-password`, `reset-password` | Public / User |
| Children | `GET /api/children/mine`, `POST /api/children`, `GET/PATCH /api/children/:id`, `GET /:id/history`, `GET /:id/audit` | Phụ huynh (bác sĩ chỉ xem trẻ đã được chỉ định) |
| Intake | `GET /api/children/frontdesk?email=...`, `POST /api/children/intake` | Lễ tân |
| Catalog | `GET /api/catalog/services`, `doctors`, `shifts`, `slots?doctorId=&date=&serviceId=` | User |
| Booking | `GET/POST /api/appointments`, `PATCH /:id/cancel`, `PATCH /:id/reschedule` | Phụ huynh/lễ tân tùy hành động |
| Reception | `PATCH /api/appointments/:id/status`, `PATCH /:id/delay` | Lễ tân, bác sĩ (bắt đầu khám), quản lý |
| Doctor | `POST /api/appointments/:id/complete` | Bác sĩ được chỉ định |
| Proposal | `POST /api/appointments/:id/proposal-response`, `POST /:id/review` | Phụ huynh |
| Messaging | `GET/POST /api/collaboration/messages/:appointmentId`, `GET /notifications`, `PATCH /notifications/:id/read` | Người dùng liên quan |
| Shifts | `POST /api/catalog/shifts`, `POST /api/manage/shift-requests`, `PATCH /api/manage/shift-requests/:id` | Quản lý hoặc bác sĩ/lễ tân |
| Manager | `POST/PATCH/PUT/DELETE /api/catalog/services`, `PATCH /api/catalog/settings`, `GET /api/manage/reports` | Quản lý |
| Admin | `GET/POST/PATCH /api/admin/users`, `GET /api/admin/audit` | Admin |
| Backup | `GET/POST /api/admin/backups`, `POST /api/admin/backups/restore` | Admin + xác nhận mật khẩu khi restore |

**Ví dụ đăng ký:**

```http
POST /api/auth/register
Content-Type: application/json

{"name":"Nguyễn Văn A","email":"phuhuynh@example.com","password":"DemoPass123!"}
```

**Ví dụ tạo lịch:**

```http
POST /api/appointments
Authorization: Bearer <accessToken>
Content-Type: application/json

{"childId":"<mongoId>","doctorId":"<mongoId>","serviceId":"<mongoId>","date":"2026-12-02","time":"08:00"}
```

## 4. Đối chiếu 28 yêu cầu chức năng trong báo cáo Word

| Mã | Triển khai trong mã nguồn |
|---|---|
| R01–R02 | API khung giờ trống, đặt lịch; unique partial MongoDB index chặn double booking |
| R03 | Đổi/hủy theo chính sách quản lý cấu hình |
| R04–R05 | Tra cứu lịch, trạng thái, check-in nhanh |
| R06 | Walk-in cho phụ huynh **đã có tài khoản**; lễ tân tìm email và tạo hồ sơ bé nếu được đồng ý |
| R07 | Lễ tân nhập số phút trễ; API cập nhật thời gian dự kiến các lịch bị ảnh hưởng (mô phỏng đơn giản) |
| R08–R09 | Bác sĩ hoàn tất khám, lưu chẩn đoán/đơn thuốc mã hóa; phụ huynh xem lịch sử khám |
| R10–R12 | Khai báo bệnh nền có xác nhận đồng ý, lễ tân cập nhật theo yêu cầu, medical audit |
| R13 | Bác sĩ tra cứu trẻ **có lịch với mình**, hiển thị cảnh báo dị ứng |
| R14–R16 | Ca làm việc, yêu cầu nghỉ/đổi ca, quản lý phê duyệt và lọc lịch bị ảnh hưởng |
| R17 | Gửi đề xuất và nhận chấp nhận/từ chối; **chưa tự động tối ưu tìm ca/bác sĩ thay thế** |
| R18–R19 | Email SMTP nếu có cấu hình; thông báo trong ứng dụng; nhắc lịch theo bộ hẹn giờ tối giản |
| R20–R21 | Nhắn tin sau khám, đánh giá 1–5 sao |
| R22–R24 | Đăng ký/đăng nhập/refresh/logout/reset, phân quyền 5 vai trò, một phụ huynh – nhiều con |
| R25–R26 | Quản lý dịch vụ, giá, chính sách, thống kê màn hình (không export) |
| R27 | Nhật ký hành động admin và thay đổi quan trọng |
| R28 | Bản sao lưu AES-GCM, backup hằng ngày khi có khóa; restore yêu cầu admin, mật khẩu, chuỗi `RESTORE` và MongoDB **replica set** |

## 5. Những hạng mục CHƯA đủ điều kiện nghiệm thu theo NF01–NF12

- **NF01 / NF02:** Chưa benchmark tải 100 user; thông báo email và thời gian phản hồi chưa có cam kết SLA. Cron in-process không đáng tin khi web app ngủ hoặc restart.
- **NF03 / NF04:** Có bcrypt, JWT + refresh rotation, timeout không hoạt động 30 phút, mã hóa AES-256-GCM, least-privilege và xác nhận consent. Tuy nhiên chưa có threat model, DPA/chính sách quyền riêng tư hoàn chỉnh, kiểm toán bảo mật, CSRF review, penetration test; demo lưu token trong `sessionStorage` (vẫn có rủi ro XSS), production nên dùng HttpOnly Secure cookie, HTTPS/TLS, CSRF và bảo vệ riêng quyền truy cập nhạy cảm.
- **NF05 / NF06:** Responsive CSS và giao diện tiếng Việt; chưa kiểm thử UAT và ma trận trình duyệt thực tế.
- **NF07:** MongoDB unique index bảo đảm không có 2 **lịch đang giữ chỗ** cùng bác sĩ + ngày + slot. Slot 30 phút cố định trong MVP. Các tác động đồng thời về ca trực, đổi ca và đặt lịch cần kiểm thử tích hợp. Dịch vụ lưu `priceAtBooking` để không đổi giá cũ.
- **NF08:** Cấu hình ngưỡng hủy/đổi và báo nghỉ. Một số tình huống hiếm như xử lý đồng thời đề xuất chuyển ca vẫn cần nghiệp vụ bổ sung.
- **NF09:** Không gửi SMS thật. SMTP do người sử dụng cấu hình, chưa có hàng đợi email bền vững.
- **NF10:** Mã nguồn phân lớp, nhóm route/middleware/model; chưa có đo lường mở rộng theo tải.
- **NF11:** Backup demo lưu **trên disk của server** (không bền vững nếu host dùng ephemeral disk), khôi phục chỉ trên replica set; chưa có PITR hay kiểm thử disaster recovery. Sao lưu tự động chỉ chạy nếu server đang chạy và đã cấu hình `BACKUP_ENCRYPTION_KEY`.
- **NF12:** Có dữ liệu thống kê; chưa đo trực tiếp giảm 50% thao tác lễ tân qua bài kiểm thử UAT.

**Phân biệt trạng thái triển khai:** Các bảng trên mô tả phần **đã viết mã nguồn**, không khẳng định tất cả mã đã chạy và vượt kiểm thử đầu-cuối. Cần chạy `npm install`, `npm test`, kiểm thử MongoDB và người dùng thực tế theo từng mã R/NF. Không triển khai cho bệnh nhân thật trước khi hoàn thiện các mục bảo mật và pháp lý.

Có sẵn file `clinic.postman_collection.json` để import Postman và thử các API chính.

## 6. Test

```bash
npm test
```

Kiểm thử thủ công quan trọng:

1. Hai request đặt cùng bác sĩ/ngày/slot → 1 thành công, 1 HTTP 409.
2. Phụ huynh A truy vấn hồ sơ bé của phụ huynh B → 403.
3. Lễ tân truy vấn kết quả khám hoặc đơn thuốc → 403.
4. Admin truy vấn lịch sử khám theo ID → 403 (không có quyền dữ liệu y tế).
5. Bác sĩ A truy vấn hồ sơ bé chỉ khám với B → 403.
6. Hủy/đổi lịch cách giờ hẹn dưới ngưỡng → 400.
7. Khóa tài khoản người dùng → token truy cập thất bại 401.
8. Refresh token cũ sau khi rotation → 401.
9. Cấu hình ca bác sĩ, đặt lịch và cập nhật trạng thái toàn vòng đời.
10. Backup khôi phục trên replica set sau khi chủ động đưa hệ thống vào thời gian bảo trì.

## 7. Triển khai tham khảo

- **Backend**: Render Web Service, root directory `server`, build `npm ci` (khi đã có lockfile), start `npm start`. Đặt tất cả biến môi trường trên dashboard (không upload file `.env`). Kết nối MongoDB Atlas. CORS `CLIENT_ORIGIN` là URL frontend chính xác. Trong production, bảo đảm HTTPS.
- **Frontend**: Render Static Site hoặc Vercel. Root `client`, build `npm run build`, publish `dist`. Set `VITE_API_URL=https://your-server.example/api` trước khi build. Với SPA, cấu hình rewrite `/* -> /index.html` (nếu nhà cung cấp hỗ trợ).
- Sao lưu quan trọng phải đưa sang hệ thống lưu trữ bền vững / Atlas Backup, không dựa trên đĩa cục bộ ephemeral của Render.

**Bản quyền / mục đích:** Project mẫu phục vụ học tập nhóm 16; các thông tin vận hành trong báo cáo có yếu tố giả định, cần xác minh trước khi sử dụng thực tế.


## 8. Cập nhật và kiểm thử theo barem

- Chạy `npm test` trong thư mục gốc để chạy bộ kiểm thử offline.
- Khi cài bản cập nhật, cần chạy lại `npm --prefix client install` vì thêm thư viện `formik` và `yup`.
- Đọc `BAREM_FIX_REPORT.md` trước khi demo. Giới hạn bảo mật và UAT vẫn áp dụng như trên.
#   b e n h - v i e n  
 