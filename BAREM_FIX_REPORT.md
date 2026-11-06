# ĐỐI CHIẾU & SỬA MÃ THEO BAREM WEB

**Nguồn barem:** `[Barem][Hiện tại] WEB Tiêu chí.xlsx`, sheet **Hướng dẫn**, các ô **C15:E28**. **Bản nguồn code:** `PhongKhamNhiHaNoi_MERN_da_sua_PostCSS.zip`. **Mục tiêu:** nâng mức đáp ứng về kỹ thuật, không cam kết điểm do BGK quyết định. Báo cáo này dựa vào *mã nguồn* và bài kiểm thử offline; chưa có kiểm thử HTTP/MongoDB end-to-end.

## 1. Tiêu chí Backend

| Mức | Barem | Bằng chứng trong mã nguồn mới | Kết luận |
|---|---|---|---|
| 4 | Node.js, npm, Express, GET/POST, middleware, truy vấn cơ bản | `server/package.json`, `server/src/index.js`, `server/src/routes/*` | Đã có mã |
| 4 | Thiết kế quan hệ CSDL 1-1, 1-n, n-n | `server/src/models/index.js` dùng Mongoose `ref` 1-n (User–Child–Appointment–Review); **chưa mô hình hóa rõ mọi loại quan hệ** | Một phần |
| 7 | PUT/DELETE, URL params | PATCH/POST/GET và **PUT/DELETE** `/api/catalog/services/:id`, URL params; DELETE chỉ ẩn dịch vụ | Đã có mã |
| 7 | Middleware validation/file/auth | Zod + JWT RBAC; không có upload file trong MVP (không phải yêu cầu cốt lõi báo cáo) | Một phần |
| 7 | Database CRUD | MongoDB, Mongoose, tạo/sửa/lấy và hủy logic | Đã có mã |
| 7 | Quản lý lỗi | `errors.js`, middleware lỗi chung trong `server/src/index.js` | Đã có mã |
| 10 | Tối ưu quản lý dữ liệu | `GET /api/appointments` phân trang, lọc, sắp xếp, giới hạn `pageSize`, chỉ mục `Appointment/Shift/Message/Notification` | Đã có mã; chưa benchmark |
| 10 | Bảo mật HTTPS/JWT/OAuth | JWT riêng access/refresh, `jti`, rotation compare-and-swap, thu hồi `sessionVersion`, bcrypt, CORS, helmet, rate-limit, encryption; HTTPS chỉ ở host, chưa OAuth | Một phần |
| 10 | MVC/RESTful/Microservices | `models/`, `routes/`, `middleware/`, `services/`, thêm `controllers/reports.controller.js`; REST API; chưa phải microservices | Một phần |
| 10 | Mô tả API | `README.md`, `clinic.postman_collection.json`, thêm `docs/API_OPENAPI.yaml` | Đã có mã |

## 2. Tiêu chí ReactJS

| Mức | Barem | Bằng chứng mới | Kết luận |
|---|---|---|---|
| 4 | Tạo React, component/JSX/props, event handlers, list keys, controlled form | `client/src/main.jsx`, `components/`, `pages/` | Đã có mã |
| 7 | `useState`, `useEffect`, Context | `client/src/context/Auth.jsx`, `components/UI.jsx` | Đã có mã |
| 7 | HOCs | `withRole` trong `client/src/main.jsx` kiểm soát quyền trang | Đã bổ sung |
| 7 | `React.memo`, `useMemo` | `Status` sử dụng `memo`, `Booking` lọc bác sĩ với `useMemo` | Đã bổ sung |
| 7 | Formik/Yup | `client/src/pages/AuthPages.jsx` dùng Formik, Yup validation đăng ký/đăng nhập/reset | Đã bổ sung (cần npm install) |
| 7 | Error Handling & Error Boundaries | `components/ErrorBoundary.jsx`, `LoadAlert` với nút thử lại | Đã bổ sung |
| 10 | Advanced Routing và Code Splitting | `React.lazy`, `Suspense`, React Router, route `/consultation/:id`, HOC bảo vệ route | Đã bổ sung |
| 10 | State Persistence | Session token dùng `sessionStorage` (giữ khi refresh cùng tab); tùy chọn dịch vụ không nhạy cảm dùng `usePreference` với localStorage | Một phần: không tự giữ đăng nhập khi đóng tab, chủ đích bảo mật |
| 10 | Class component, lifecycle | `ErrorBoundary` dùng `getDerivedStateFromError`, `componentDidCatch` | Đã bổ sung; tối ưu render bằng memo ở component khác |

## 3. Lỗi đã sửa

- Không chấp nhận **24:00/25:00** hoặc ngày không tồn tại **30/02**; ngày/giờ được xác thực trước khi tạo lịch, đổi lịch, tạo ca, đổi ca, nhập hồ sơ.
- Refresh token có `jti` ngẫu nhiên nên các lần cấp lại khác nhau, rotation bằng cập nhật điều kiện trong MongoDB. Access token được thu hồi theo `sessionVersion` sau logout, đổi mật khẩu hoặc admin khóa/đổi role.
- Danh sách lịch khám có phân trang ở UI; API giới hạn kích thước trang để tránh truy vấn thiếu giới hạn.
- React xử lý lỗi tải API rõ hơn, có retry; code splitting cải thiện tải trang khi build production.

## 4. Kiểm thử và hạn chế

Chạy `npm test` tại `clinic-booking`: **26/26 bài kiểm thử offline đạt** trong môi trường tạo gói, gồm logic thời gian, mã hóa, phân quyền giả lập, kiểm tra sessionVersion, import và cú pháp JSX. **Các bài unit đạt chưa chứng minh toàn bộ API hoạt động đúng với MongoDB.**

Trong môi trường tạo gói, `npm --prefix client install` bị `EAI_AGAIN` (không truy cập được npm registry), nên chưa chạy được `vite build` và chưa thể chứng minh Formik/Yup hoạt động lúc chạy. Mọi chức năng database cần kiểm thử lại bằng MongoDB + Postman/browser. Barem là tiêu chí tham khảo, không có thuật toán bắt buộc cộng điểm.

**Chưa bổ sung (và không nên tạo hình thức chỉ để tăng điểm):** OAuth, upload file không thuộc phạm vi cốt lõi, microservices, chứng chỉ HTTPS chạy local, kiểm thử hiệu năng 100 người, UAT thực tế. Hệ thống vẫn là bản demo; không dùng dữ liệu y tế thật.

## 5. Chạy lại trên Windows

```powershell
cd "C:\\Users\\Admin\\Downloads\\TEN_THU_MUC_GIAI_NEN\\clinic-booking"
npm install
npm --prefix server install
npm --prefix client install
npm --prefix client run build
npm test
```

Sau khi cấu hình `server/.env` và `client/.env` (theo `README.md`) và bật MongoDB:

```powershell
npm --prefix server run seed
npm run dev
```

- Frontend: `http://localhost:5173`
- API: `http://localhost:8080/api/health`
- OpenAPI: `docs/API_OPENAPI.yaml` (có thể import vào Swagger Editor)
- Postman: `clinic.postman_collection.json`
