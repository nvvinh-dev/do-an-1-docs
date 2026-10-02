# Danh sách màn hình

Toàn bộ màn hình frontend của Phase 1: đường dẫn, file trang, API gọi tới, thao tác theo trạng thái và người làm. Body, response và mã lỗi của từng API xem ở [Thiết kế API](api-design.md); thứ tự và ngày làm xem ở bảng tiến độ của nhóm.

Mỗi màn hình là một file trong `frontend/src/pages/<vai trò>/` (README mục 10). Đường dẫn viết bằng tiếng Anh, theo đúng tên thư mục vai trò (`/tenant`, `/landlord`, `/admin`) và tên tài nguyên của API, để đường dẫn, tên file và API không lệch nhau. Trang có tên trùng giữa hai vai trò thì thêm tiền tố vai trò vào tên file, ví dụ `TenantContractDetailPage.tsx` và `LandlordContractDetailPage.tsx`.

---

## 1. Quy ước chung

**Layout theo vai trò** (`layouts/`): `PublicLayout` cho khách, `TenantLayout`, `LandlordLayout`, `AdminLayout`. Layout đã đăng nhập có menu theo vai trò, chuông thông báo (số chưa đọc lấy từ `GET /notifications/unread-count`), liên kết tới trang tài khoản và nút đăng xuất.

**Phân quyền route:**

- `/tenant/*` chỉ cho Tenant, `/landlord/*` chỉ cho Landlord, `/admin/*` chỉ cho Admin. Chưa đăng nhập thì chuyển tới `/login?redirect=<đường dẫn đang mở>`; sai vai trò thì về trang chủ của vai trò mình.
- Đăng nhập xong: Tenant về `/tenant`, Landlord về `/landlord`, Admin về `/admin` — hoặc về `redirect` nếu có.
- Ẩn nút và chặn route ở frontend chỉ để trải nghiệm; mọi kiểm soát thật nằm ở backend (Kiến trúc mục 8).

**Hiển thị lỗi từ API** (ProblemDetails, API mục 1):

| Mã | Cách hiển thị |
|---|---|
| `401` | `lib/api.ts` xóa token và chuyển tới `/login` |
| `403` | Thông báo "Bạn không có quyền thực hiện thao tác này" |
| `404` | Trang `NotFoundPage` — tài nguyên không tồn tại hoặc không thuộc người gọi |
| `409`, `422` | Hiện `detail` ngay gần nút vừa bấm hoặc ở đầu form; không xóa dữ liệu người dùng đã nhập |
| `429` | Hiện `detail` và thời gian chờ theo header `Retry-After` |
| Lỗi mạng, `500` | Thông báo chung kèm nút thử lại |

**Trạng thái của mọi trang có dữ liệu:** đang tải (khung xương hoặc vòng xoay), rỗng (câu hướng dẫn và nút hành động nếu có), lỗi (theo bảng trên).

**Quy ước khác:**

- Thao tác không hoàn tác — hủy hợp đồng, hoàn tất thanh lý, tự chốt bảng thanh lý, khóa tài khoản, lưu trữ khu trọ hoặc phòng — có hộp thoại xác nhận trước khi gửi (FR-75).
- Tiền định dạng bằng `utils/formatCurrency.ts` (ví dụ `1.645.161 ₫`), ngày dạng `dd/mm/yyyy` theo giờ Việt Nam. Mã trạng thái (`ChoDuyet`, `DangHieuLuc`...) hiển thị bằng nhãn tiếng Việt có dấu, gom ở một file trong `utils/`.
- Hạn có mốc giờ (`expiresAt`, `holdExpiresAt`) hiển thị dạng đếm ngược "còn X giờ".
- Ảnh và giấy tờ tải lên qua `POST /files` với đúng `purpose`, có xem trước; dung lượng tối đa 5 MB với ảnh công khai, 10 MB với file riêng tư (API mục 14).
- Mã VietQR vẽ trong trình duyệt từ trường `paymentQr` (Kiến trúc mục 8); `paymentQr` là `null` thì không hiện khung mã.
- Dùng được trên màn hình rộng từ 360px.

---

## 2. Khách — `pages/public/`

| Mã | Màn hình | Đường dẫn | File trang | API | Người làm |
|---|---|---|---|---|---|
| P1 | Tìm phòng (trang chủ) | `/` | `SearchPage.tsx` | `GET /rooms/search`, `GET /locations`, `GET /amenities` | Vinh |
| P2 | Chi tiết phòng | `/rooms/:id` | `RoomPublicDetailPage.tsx` | `GET /rooms/{id}/public` | Vinh |
| P3 | Không tìm thấy | `*` | `NotFoundPage.tsx` | — | Giang |

- **P1:** bộ lọc tỉnh/thành → phường/xã (chỉ chọn phường sau khi đã chọn tỉnh), khoảng giá, khoảng diện tích, tiện ích, số người; sắp xếp; phân trang. Bộ lọc nằm trên query của URL để quay lại hoặc chia sẻ không mất kết quả.
- **P2:** ảnh theo thứ tự, giá, đơn giá điện nước, phí dịch vụ, tiện ích của phòng và khu trọ. Người thuê đã đăng nhập thấy nút **Gửi yêu cầu thuê** mở form `RentalRequestForm` (T3); khách thấy **Đăng nhập để gửi yêu cầu**; Chủ trọ và Admin không thấy nút. Phòng không đủ điều kiện BR-05 trả `404` → `NotFoundPage`.

---

## 3. Xác thực — `pages/auth/`

| Mã | Màn hình | Đường dẫn | File trang | API | Người làm |
|---|---|---|---|---|---|
| A1 | Đăng nhập | `/login` | `LoginPage.tsx` | `POST /auth/login` | Giang |
| A2 | Đăng ký | `/register` | `RegisterPage.tsx` | `POST /auth/register` | Giang |
| A3 | Quên mật khẩu | `/forgot-password` | `ForgotPasswordPage.tsx` | `POST /auth/forgot-password` | Giang |
| A4 | Đặt lại mật khẩu | `/reset-password` | `ResetPasswordPage.tsx` | `POST /auth/reset-password` | Giang |

- **A1:** `401` → "Email hoặc mật khẩu không đúng"; `403` → tài khoản bị khóa, kèm lý do; `429` → báo thời gian phải chờ.
- **A3:** luôn hiện cùng một thông báo, dù email có tồn tại hay không.
- **A4:** đọc `email` và `token` từ query của đường dẫn trong email (`/reset-password?email=...&token=...`, API mục 2).

---

## 4. Dùng chung — `pages/shared/`

| Mã | Màn hình | Đường dẫn | File trang | API | Người làm |
|---|---|---|---|---|---|
| S1 | Tài khoản | `/account` | `AccountPage.tsx` | `GET /auth/me`, `PUT /auth/me`, `POST /auth/change-password` | Giang |
| S2 | Thông báo | `/notifications` | `NotificationsPage.tsx` | `GET /notifications`, `POST /notifications/{id}/read`, `POST /notifications/read-all` | Giang |

- **S1:** sửa họ tên, số điện thoại; đổi mật khẩu. Đổi số điện thoại thì số bị bỏ trạng thái đã xác thực (BR-01) — hiện cảnh báo trước khi lưu.
- **S2:** lọc đã đọc / chưa đọc. Bấm một thông báo thì đánh dấu đã đọc và đi tới trang tương ứng theo `relatedEntityType` và vai trò hiện tại: `Contract` → trang hợp đồng, `Invoice` → trang hóa đơn, `RentalRequest` → trang yêu cầu thuê, `LandlordApplication` → hồ sơ Chủ trọ, `AppUser` → trang tài khoản.

---

## 5. Người thuê — `pages/tenant/`

| Mã | Màn hình | Đường dẫn | File trang | API | Người làm |
|---|---|---|---|---|---|
| T1 | Dashboard | `/tenant` | `TenantDashboardPage.tsx` | `GET /dashboard/tenant` | Giang |
| T2 | Đăng ký làm Chủ trọ | `/tenant/landlord-application` | `LandlordApplicationPage.tsx` | `GET /landlord-applications/me`, `POST /files` (`GiayToNhanThan`), `POST /landlord-applications` | Giang |
| T3 | Yêu cầu thuê của tôi | `/tenant/rental-requests`, `/tenant/rental-requests/:id` | `TenantRentalRequestListPage.tsx`, `TenantRentalRequestDetailPage.tsx`; component `RentalRequestForm` | `GET /rental-requests`, `GET /rental-requests/{id}`, `POST /rooms/{roomId}/rental-requests`, `POST /rental-requests/{id}/cancel` | Giang |
| T4 | Hợp đồng của tôi | `/tenant/contracts`, `/tenant/contracts/:id` | `TenantContractListPage.tsx`, `TenantContractDetailPage.tsx` | `GET /contracts`, `GET /contracts/{id}`, `/confirm`, `/request-changes`, `/cancel`, `/move-out-notice`, `/move-out-notice/withdraw` | Giang |
| T5 | Hóa đơn của tôi | `/tenant/invoices`, `/tenant/invoices/:id` | `TenantInvoiceListPage.tsx`, `TenantInvoiceDetailPage.tsx` | `GET /invoices`, `GET /invoices/{id}`, `POST /files` (`MinhChungThanhToan`), `POST /invoices/{id}/payment-reports` | Vinh |
| T6 | Bảng thanh lý | Trong T5, khi hóa đơn có `type = ThanhLy` | component `SettlementReview` | `POST /contracts/{id}/settlement-invoice/confirm`, `/request-changes` | Giang |

**T1:** hợp đồng hiện tại, hóa đơn chưa thanh toán, tổng đã thanh toán; mục **Việc cần xử lý** (`pendingActions`) dẫn tới T4 lọc hợp đồng chờ xác nhận và T5 lọc bảng thanh lý chờ đồng ý.

**T2:** chưa nộp → form tải 3 ảnh giấy tờ và số CCCD; đang chờ duyệt → hiện trạng thái; bị từ chối → hiện lý do và nút nộp lại. Tài khoản chưa có số điện thoại, hoặc còn hợp đồng hay yêu cầu thuê đang mở, thì hiện `detail` của lỗi trả về.

**T3 — thao tác theo trạng thái:**

| Trạng thái | Hiển thị và thao tác |
|---|---|
| `ChoDuyet` | Đếm ngược `expiresAt`; nút **Rút yêu cầu** |
| `DaDuyet` | Số điện thoại Chủ trọ; đếm ngược `holdExpiresAt`; nút **Rút yêu cầu** |
| `DaLapHopDong` | Liên kết tới hợp đồng (`contractId`) |
| `TuChoi`, `DaHuy`, `HetHan` | Chỉ xem; hiện lý do từ chối nếu có |

`RentalRequestForm`: ngày dự kiến vào ở (không trước hôm nay), số người (1 tới số người tối đa của phòng), ghi chú.

**T4 — thao tác theo trạng thái:**

| Trạng thái | Hiển thị và thao tác |
|---|---|
| `Nhap` | Chỉ đọc, nhãn "Chủ trọ đang soạn"; nút **Hủy hợp đồng** |
| `ChoNguoiThueXacNhan` | Đủ điều khoản, phí dịch vụ, người ở cùng, chỉ số đầu; nút **Đồng ý**, **Yêu cầu sửa** (bắt buộc lý do), **Hủy hợp đồng** |
| `ChoNhanCoc` | Mã VietQR tiền cọc; đếm ngược `holdExpiresAt`; nút **Hủy hợp đồng** |
| `DangHieuLuc` trước ngày bắt đầu | Nút **Hủy hợp đồng**, cảnh báo có thể mất cọc (BR-22) |
| `DangHieuLuc`, `SapHetHan` từ ngày bắt đầu | Nút **Gửi thông báo trả phòng**; sau khi gửi hiện `noticeDays` và `penaltyAllowed` |
| `DangThanhLy` | Thông tin thông báo trả phòng; nút **Rút thông báo** khi mình là bên gửi và chưa có bảng thanh lý; liên kết tới hóa đơn thanh lý khi đã có |
| `DaHuy` | Lý do hủy; thông tin hoàn cọc (`depositRefund`) hoặc nhãn "Chờ hoàn cọc" |
| `DaThanhLy` | Chỉ xem |

**T5 — thao tác theo trạng thái:**

| Trạng thái | Hiển thị và thao tác |
|---|---|
| `ChuaThanhToan`, `ThanhToanMotPhan`, `QuaHan` | Mã VietQR theo số còn phải trả; nút **Báo đã thanh toán** (số tiền và ảnh minh chứng) |
| `ChoXacNhan` | Lượt báo đang chờ Chủ trọ xác nhận |
| `ChoNguoiThueXacNhan` (hóa đơn thanh lý) | `SettlementReview`: từng dòng, chiều số dư; nút **Đồng ý**, **Chưa đồng ý** (bắt buộc lý do) |
| Mọi trạng thái | Cách tính: chỉ số cũ và mới, đơn giá, `daysCharged`/`daysInMonth`, phí dịch vụ, các dòng điều chỉnh; lịch sử lượt báo thanh toán |

---

## 6. Chủ trọ — `pages/landlord/`

| Mã | Màn hình | Đường dẫn | File trang | API | Người làm |
|---|---|---|---|---|---|
| L1 | Dashboard | `/landlord` | `LandlordDashboardPage.tsx` | `GET /dashboard/landlord` | Vinh |
| L2 | Khu trọ | `/landlord/properties`, `/landlord/properties/new`, `/landlord/properties/:id`, `/landlord/properties/:id/edit` | `PropertyListPage.tsx`, `PropertyFormPage.tsx`, `PropertyDetailPage.tsx` | `GET /properties`, `POST /properties`, `GET /properties/{id}`, `PUT /properties/{id}`, `POST /properties/{id}/archive`, `GET /properties/{propertyId}/rooms`, `GET /locations`, `GET /amenities`, `POST /files` (`AnhKhuTro`) | Vinh |
| L3 | Phòng | `/landlord/properties/:propertyId/rooms/new`, `/landlord/rooms/:id`, `/landlord/rooms/:id/edit` | `RoomFormPage.tsx`, `LandlordRoomDetailPage.tsx` | `POST /properties/{propertyId}/rooms`, `GET /rooms/{id}`, `PUT /rooms/{id}`, `PATCH /rooms/{id}/visibility`, `PATCH /rooms/{id}/occupancy-status`, `POST /rooms/{id}/archive`, `POST /files` (`AnhPhong`) | Vinh |
| L4 | Tài khoản nhận tiền | `/landlord/bank-account` | `BankAccountPage.tsx` | `GET /landlord/bank-account`, `PUT /landlord/bank-account` | Vinh |
| L5 | Yêu cầu thuê | `/landlord/rental-requests`, `/landlord/rental-requests/:id` | `LandlordRentalRequestListPage.tsx`, `LandlordRentalRequestDetailPage.tsx` | `GET /rental-requests`, `GET /rental-requests/{id}`, `/approve`, `/reject` | Vinh |
| L6 | Hợp đồng | `/landlord/contracts`, `/landlord/contracts/new?rentalRequestId=`, `/landlord/contracts/:id`, `/landlord/contracts/:id/edit` | `LandlordContractListPage.tsx`, `ContractFormPage.tsx`, `LandlordContractDetailPage.tsx` | `GET /contracts`, `POST /contracts`, `GET /contracts/{id}`, `PUT /contracts/{id}`, `/send`, `/recall`, `/deposit/confirm`, `/cancel`, `/deposit/refund`, `PATCH /initial-meter-readings`, `GET /rooms/{id}/meter-readings/latest`, `GET /contracts/{id}/invoices`, `/move-out-notice`, `/move-out-notice/withdraw` | Vinh; phần thông báo trả phòng: Giang |
| L7 | Chốt chỉ số, lập và sửa hóa đơn | `/landlord/contracts/:id/invoices/new`, `/landlord/invoices/:id/edit` | `InvoiceFormPage.tsx` | `GET /contracts/{contractId}/meter-readings/latest`, `POST /contracts/{contractId}/invoices`, `PUT /invoices/{id}`, `POST /files` (`AnhDongHo`) | Vinh |
| L8 | Hóa đơn | `/landlord/invoices`, `/landlord/invoices/:id` | `LandlordInvoiceListPage.tsx`, `LandlordInvoiceDetailPage.tsx` | `GET /invoices`, `GET /invoices/{id}`, `/issue`, `/cancel`, `/payment-reports/{reportId}/confirm`, `/payment-reports/{reportId}/reject` | Vinh |
| L9 | Thanh lý | `/landlord/contracts/:id/settlement` | `SettlementPage.tsx` | `POST /contracts/{id}/settlement-invoice`, `PUT /contracts/{id}/settlement-invoice`, `/send`, `/finalize`, `POST /contracts/{id}/settlement/complete`, `GET /invoices/{id}`, `POST /files` (`AnhHuHong`, `AnhDongHo`) | Giang |

**L1:** số phòng theo trạng thái, hóa đơn chưa thu, biểu đồ doanh thu 6 tháng; mục **Việc cần xử lý** dẫn tới L5 lọc `ChoDuyet`, L8 lọc `ChoXacNhan`, L6 lọc `ChoNhanCoc`.

**L2:** danh sách khu trọ kèm số phòng theo trạng thái, mặc định ẩn khu đã lưu trữ (`includeArchived=true` để xem cả). Form chọn tỉnh/thành → phường/xã từ `GET /locations`, tiện ích từ `GET /amenities` (phạm vi khu trọ), tối đa 10 ảnh và sắp được thứ tự. Lưu trữ khi còn phòng đang giữ chỗ hoặc đang thuê thì hiện `detail` (`409`).

**L3:** form giá, đơn giá điện nước, diện tích, số người tối đa, phí dịch vụ, tiện ích (phạm vi phòng), tối đa 10 ảnh. Bật hiển thị khi phòng chưa có ảnh trả `422` → hiện `detail`. Chủ trọ chỉ tự chuyển phòng giữa Trống và Bảo trì; nút lưu trữ chỉ hiện với phòng Trống hoặc Bảo trì.

**L4:** chọn ngân hàng theo danh sách VietQR, số tài khoản, tên chủ tài khoản viết hoa không dấu. Chưa khai báo thì hiện lời nhắc: người thuê sẽ không thấy mã VietQR.

**L5 — thao tác theo trạng thái:**

| Trạng thái | Hiển thị và thao tác |
|---|---|
| `ChoDuyet` | Đếm ngược `expiresAt`; nút **Duyệt**, **Từ chối** (bắt buộc lý do). Người thuê đang giữ phòng khác thì `409` → hiện `detail` |
| `DaDuyet` | Số điện thoại người thuê; đếm ngược `holdExpiresAt`; nút **Lập hợp đồng** (→ L6), **Hủy duyệt** (bắt buộc lý do) |
| `DaLapHopDong` | Liên kết tới hợp đồng |
| `TuChoi`, `DaHuy`, `HetHan` | Chỉ xem |

**L6 — form lập hợp đồng:** điền sẵn giá thuê, đơn giá, phí dịch vụ từ phòng (`GET /rooms/{id}`) và chỉ số đầu từ `GET /rooms/{id}/meter-readings/latest`; người ở cùng; kiểm tra tổng số người không vượt số người tối đa; ngày bắt đầu không trước hôm nay.

**L6 — thao tác theo trạng thái:**

| Trạng thái | Hiển thị và thao tác |
|---|---|
| `Nhap` | Nút **Sửa**, **Gửi cho người thuê**, **Hủy hợp đồng** |
| `ChoNguoiThueXacNhan` | Nút **Thu hồi để sửa**, **Hủy hợp đồng** |
| `ChoNhanCoc` | Nút **Xác nhận đã nhận cọc** (ngày nhận, hình thức `TienMat` / `ChuyenKhoan`), **Thu hồi để sửa**, **Hủy hợp đồng** |
| `DangHieuLuc` trước ngày bắt đầu | Nút **Hủy hợp đồng** (hoàn toàn bộ cọc) |
| `DangHieuLuc` chưa có hóa đơn | Nút **Sửa chỉ số đầu** |
| `DangHieuLuc`, `SapHetHan` từ ngày bắt đầu | Danh sách hóa đơn của hợp đồng; nút **Lập hóa đơn kỳ tiếp** (→ L7), **Gửi thông báo trả phòng** |
| `DangThanhLy` | Thông tin thông báo trả phòng; nút **Rút thông báo** khi mình là bên gửi và chưa có bảng thanh lý; **Lập hóa đơn định kỳ** cho tháng trọn còn thiếu; **Lập bảng thanh lý** (→ L9) từ ngày trả phòng |
| `DaHuy` đã nhận cọc, chưa hoàn | Nút **Ghi nhận hoàn cọc** — người thuê hủy thì nhập số hoàn từ 0 tới tiền cọc, kèm lý do khi giữ lại |
| `DaThanhLy`, `DaHuy` đã hoàn cọc | Chỉ xem |

**L7:** hiện chỉ số cũ do hệ thống điền, nhập chỉ số mới và ảnh đồng hồ, thêm dòng điều chỉnh (mô tả, số tiền, hóa đơn gốc); xem trước tổng tiền trước khi lưu. Kỳ chưa tới, hoặc còn hóa đơn nháp, thì `409` → hiện `detail`.

**L8 — thao tác theo trạng thái** (danh sách lọc theo trạng thái, tháng, khu trọ):

| Trạng thái | Hiển thị và thao tác |
|---|---|
| `Nhap` | Nút **Sửa** (→ L7), **Phát hành**, **Hủy** |
| `ChuaThanhToan`, `QuaHan` chưa thu đồng nào | Nút **Sửa**, **Hủy** — chỉ với hóa đơn định kỳ mới nhất của hợp đồng |
| `ChoXacNhan` | Lượt báo đang chờ: ảnh minh chứng, số tiền người thuê khai; nút **Xác nhận** (số tiền thực thu), **Từ chối** (bắt buộc lý do) |
| Mọi trạng thái | Cách tính và lịch sử lượt báo thanh toán như T5 |

**L9 — thao tác theo trạng thái:**

| Trạng thái | Hiển thị và thao tác |
|---|---|
| Chưa có bảng | Form lập: chỉ số cuối và ảnh đồng hồ, ngày trả phòng thực tế, dòng bồi thường hư hỏng kèm ảnh, dòng phí phạt (chỉ hiện khi `penaltyAllowed`), dòng điều chỉnh. Dòng công nợ cũ và dòng trừ tiền cọc do hệ thống thêm, chỉ hiển thị |
| `Nhap` | Nút **Sửa**, **Gửi cho người thuê**; hiện lý do chưa đồng ý gần nhất nếu có |
| `ChoNguoiThueXacNhan` | Chờ người thuê; đủ 7 ngày từ `sentAt` thì hiện nút **Tự chốt** (bắt buộc ghi chú) |
| Đã khóa | Nút **Hoàn tất thanh lý**: chọn phòng về Trống hoặc Bảo trì; số dư âm thì nhập ngày và hình thức hoàn cọc |

Số dư luôn hiện rõ chiều: dương là người thuê còn phải trả, âm là Chủ trọ phải hoàn.

---

## 7. Admin — `pages/admin/`

| Mã | Màn hình | Đường dẫn | File trang | API | Người làm |
|---|---|---|---|---|---|
| AD1 | Dashboard | `/admin` | `AdminDashboardPage.tsx` | `GET /dashboard/admin` | Giang |
| AD2 | Hồ sơ Chủ trọ | `/admin/landlord-applications`, `/admin/landlord-applications/:id` | `LandlordApplicationListPage.tsx`, `LandlordApplicationReviewPage.tsx` | `GET /admin/landlord-applications`, `GET /admin/landlord-applications/{id}`, `/approve`, `/reject` | Giang |
| AD3 | Tài khoản | `/admin/users` | `UserListPage.tsx` | `GET /admin/users`, `POST /admin/users/{id}/lock`, `POST /admin/users/{id}/unlock` | Giang |
| AD4 | Nhật ký | `/admin/audit-logs` | `AuditLogPage.tsx` | `GET /admin/audit-logs` | Giang |

- **AD2:** ảnh giấy tờ là URL có hạn — không lưu lại. Duyệt thì nhập số điện thoại đã gọi xác minh (`verifiedPhoneNumber`); số đã đổi thì `409` → hiện `detail`. Từ chối bắt buộc lý do.
- **AD3:** lọc theo vai trò và trạng thái khóa. Khóa bắt buộc lý do; Chủ trọ có `activeContractCount` > 0 thì cảnh báo trước khi khóa — giới hạn đã chấp nhận ở Phase 1 (API mục 4).
- **AD4:** lọc theo đối tượng, người thực hiện, loại thao tác, khoảng thời gian; `oldValue` và `newValue` hiển thị dạng bảng so sánh trước – sau.
