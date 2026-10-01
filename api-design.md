# Thiết kế API — Phase 1

Tài liệu chính thức về contract API giữa frontend và backend.

**Phạm vi:** Phase 1 theo phân kỳ trong [README](../README.md). Endpoint phục vụ sự cố, ở ghép, khiếu nại, lịch xem phòng, gia hạn hợp đồng và trợ lý AI **không** nằm trong tài liệu này.

Tài liệu này bám theo [Thiết kế Cơ sở dữ liệu](database-design.md). Tên trạng thái trả về trong response là đúng giá trị lưu trong database.

---

## 1. Quy ước chung

| Hạng mục | Quy ước |
|---|---|
| **Đường dẫn gốc** | `/api/v1` |
| **Định dạng** | JSON, `Content-Type: application/json` |
| **Tên trường** | `camelCase` |
| **Xác thực** | JWT, gửi qua header `Authorization: Bearer <token>`. Access token sống **60 phút**; hết hạn thì đăng nhập lại — hệ thống không dùng refresh token |
| **Dữ liệu trả về** | Luôn là lớp DTO riêng trong `Contracts/`, **không** serialize thẳng entity |
| **Lỗi** | `application/problem+json` theo chuẩn ProblemDetails của ASP.NET Core |
| **Phân trang** | Query `page` (bắt đầu từ 1) và `pageSize`; response bọc trong `{ items, page, pageSize, totalItems, totalPages }` |
| **Ngày giờ** | ISO 8601, múi giờ UTC |
| **Tiền** | Số, không định dạng, không kèm đơn vị |

### 1.1 Mã trạng thái

| Mã | Dùng khi |
|---|---|
| `200` | Đọc hoặc cập nhật thành công |
| `201` | Tạo mới thành công, kèm header `Location` |
| `204` | Thành công, không có nội dung trả về |
| `400` | Dữ liệu vào không hợp lệ |
| `401` | Thiếu token hoặc token không hợp lệ |
| `403` | Có token nhưng không đủ quyền với tài nguyên đó |
| `404` | Không tìm thấy, hoặc tài nguyên không thuộc quyền truy cập của người gọi |
| `409` | Vi phạm quy tắc nghiệp vụ về trạng thái |
| `422` | Vi phạm quy tắc nghiệp vụ về dữ liệu |
| `429` | Vượt ngưỡng rate limiting, kèm header `Retry-After` |

### 1.2 Nguyên tắc phân quyền

Backend kiểm tra quyền trên **mọi** request cần bảo vệ. Ẩn nút ở frontend không được coi là phân quyền. Ma trận phân quyền đầy đủ và mô hình mối đe dọa nằm ở [Thiết kế An toàn](security-design.md).

- **Danh tính lấy từ token.** `userId` và vai trò của người thao tác luôn đọc từ JWT, không bao giờ đọc từ body, query hay header tự đặt. Client gửi các giá trị này lên thì bỏ qua.
- **Từ chối khi không chắc chắn.** Thiếu token, token không hợp lệ, hoặc không xác định được quyền sở hữu vì bất kỳ lý do gì — kể cả lỗi hệ thống — đều trả về từ chối và ghi log, không cho request đi tiếp.
- **BR-04:** Chủ trọ chỉ thao tác được trên khu trọ, phòng, hợp đồng và hóa đơn thuộc khu trọ của mình. Người thuê chỉ đọc được hợp đồng và hóa đơn của chính mình.
- Khi người gọi không có quyền với một tài nguyên tồn tại, trả `404` thay vì `403` đối với các tài nguyên có thể bị dò id — hợp đồng, hóa đơn, hồ sơ Chủ trọ.
- **BR-24:** Admin không có quyền đọc mặc định đối với hợp đồng và hóa đơn. Phase 1 không có endpoint nào cho phép Admin đọc hai loại tài nguyên này.
- **QR-04:** ảnh giấy tờ nhân thân chỉ xuất hiện trong response của endpoint duyệt hồ sơ dành cho Admin.
- **BR-26:** tài khoản ngân hàng của Chủ trọ chỉ xuất hiện trong `GET /landlord/bank-account` của chính Chủ trọ đó và trong trường `paymentQr` gửi cho Người thuê đứng tên hợp đồng (Mục 9.1).

---

## 2. Xác thực — BP-01

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Công khai | Đăng ký tài khoản, mặc định nhận vai trò `Tenant` |
| `POST` | `/api/v1/auth/login` | Công khai | Đăng nhập, trả về JWT |
| `POST` | `/api/v1/auth/forgot-password` | Công khai | Gửi yêu cầu đặt lại mật khẩu |
| `POST` | `/api/v1/auth/reset-password` | Công khai | Đặt lại mật khẩu bằng token |
| `POST` | `/api/v1/auth/change-password` | Đã đăng nhập | Đổi mật khẩu |
| `GET` | `/api/v1/auth/me` | Đã đăng nhập | Thông tin tài khoản và vai trò hiện tại |
| `PUT` | `/api/v1/auth/me` | Đã đăng nhập | Cập nhật thông tin cá nhân |

**`POST /api/v1/auth/login`**

```json
{ "email": "tenant@example.com", "password": "..." }
```

```json
{
  "accessToken": "eyJhbGciOi...",
  "expiresAt": "2026-09-18T12:00:00Z",
  "user": { "id": 12, "fullName": "Nguyen Van A", "email": "tenant@example.com", "roles": ["Tenant"] }
}
```

Tài khoản có `isLocked = true` nhận `403` kèm lý do khóa — chỉ khi email và mật khẩu đúng. Sai email hoặc mật khẩu luôn nhận `401` với cùng một thông báo, để không dò được email nào tồn tại hay đang bị khóa.

Đăng nhập sai 5 lần liên tiếp trong 15 phút với cùng một cặp email + địa chỉ IP thì nhận `429` kèm `Retry-After` cho tới hết 15 phút, kể cả khi email không tồn tại (Thiết kế An toàn mục 6).

**`PUT /api/v1/auth/me`** — body gồm `fullName` và `phoneNumber`. Đổi `phoneNumber` thì trạng thái đã xác thực của số điện thoại bị bỏ (BR-01).

### 2.1 Tài khoản nhận tiền của Chủ trọ — BR-26

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/landlord/bank-account` | Landlord | Xem tài khoản nhận tiền của chính mình; chưa khai báo thì trả `null` |
| `PUT` | `/api/v1/landlord/bank-account` | Landlord | Khai báo hoặc cập nhật tài khoản nhận tiền |

**`PUT /api/v1/landlord/bank-account`**

```json
{ "bankBin": "970436", "accountNumber": "0123456789", "accountName": "NGUYEN VAN A" }
```

Cả ba trường bắt buộc. `bankBin` phải là mã BIN 6 chữ số có trong danh sách ngân hàng của chuẩn VietQR; `accountNumber` chỉ gồm chữ số; `accountName` viết hoa không dấu. Sai định dạng trả `422`. Chủ trọ chỉ đọc và sửa được tài khoản của chính mình — `userId` lấy từ token. Mỗi lần gọi `PUT` ghi `audit_logs` với giá trị cũ và mới theo BR-23.

---

## 3. Hồ sơ Chủ trọ — BP-01

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/landlord-applications` | Tenant | Nộp hồ sơ đăng ký làm Chủ trọ |
| `GET` | `/api/v1/landlord-applications/me` | Đã đăng nhập | Xem hồ sơ và trạng thái của chính mình |
| `GET` | `/api/v1/admin/landlord-applications` | Admin | Danh sách hồ sơ, lọc theo `status` |
| `GET` | `/api/v1/admin/landlord-applications/{id}` | Admin | Chi tiết hồ sơ, bao gồm ảnh giấy tờ |
| `POST` | `/api/v1/admin/landlord-applications/{id}/approve` | Admin | Duyệt, cấp vai trò `Landlord` |
| `POST` | `/api/v1/admin/landlord-applications/{id}/reject` | Admin | Từ chối, bắt buộc có `reason` |

**`POST /api/v1/landlord-applications`**

```json
{
  "idCardNumber": "079xxxxxxxxx",
  "idCardFrontPath": "private-documents/GiayToNhanThan/12/2026/09/...",
  "idCardBackPath": "private-documents/GiayToNhanThan/12/2026/09/...",
  "ownershipDocumentPath": "private-documents/GiayToNhanThan/12/2026/09/..."
}
```

Ba đường dẫn là giá trị `path` nhận được từ `POST /files` với `purpose = GiayToNhanThan`, và phải do chính người nộp tải lên (Mục 14); sai thì trả `422`. Tài khoản chưa có số điện thoại trả `422`. Đã có hồ sơ ở `ChoDuyet` trả `409`.

**`POST /approve`** — Admin xác minh số điện thoại của người nộp trước khi duyệt. Duyệt thì tài khoản được cấp vai trò `Landlord` thay cho `Tenant`, và số điện thoại được ghi nhận là đã xác thực (BR-01). Người nộp phải đăng nhập lại để token mang vai trò mới.

Duyệt và từ chối đều ghi `audit_logs` theo BR-23. Người nộp nhận thông báo mức Cao.

---

## 4. Quản lý tài khoản — BP-01

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/admin/users` | Admin | Danh sách tài khoản, lọc theo vai trò và trạng thái khóa |
| `POST` | `/api/v1/admin/users/{id}/lock` | Admin | Khóa tài khoản, bắt buộc có `reason` |
| `POST` | `/api/v1/admin/users/{id}/unlock` | Admin | Mở khóa |

Cả hai thao tác ghi `audit_logs` và gửi thông báo mức Cao cho người bị xử lý.

---

## 5. Khu trọ và phòng trọ — BP-02, BP-03

### 5.1 Khu trọ

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/properties` | Landlord | Tạo khu trọ |
| `GET` | `/api/v1/properties` | Landlord | Danh sách khu trọ của chính mình |
| `GET` | `/api/v1/properties/{id}` | Landlord (chủ sở hữu) | Chi tiết |
| `PUT` | `/api/v1/properties/{id}` | Landlord (chủ sở hữu) | Cập nhật |
| `POST` | `/api/v1/properties/{id}/archive` | Landlord (chủ sở hữu) | Lưu trữ khu trọ |

`archive` trả `409` khi còn phòng ở `DangGiuCho` hoặc `DangThue` (BR-10). Không có endpoint `DELETE` (BR-09).

### 5.2 Phòng trọ

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/properties/{propertyId}/rooms` | Landlord (chủ sở hữu) | Thêm phòng, kèm giá và phí dịch vụ |
| `GET` | `/api/v1/properties/{propertyId}/rooms` | Landlord (chủ sở hữu) | Danh sách phòng của khu trọ |
| `GET` | `/api/v1/rooms/{id}` | Landlord (chủ sở hữu) | Chi tiết phòng ở góc nhìn quản lý |
| `PUT` | `/api/v1/rooms/{id}` | Landlord (chủ sở hữu) | Cập nhật thông tin và giá |
| `PATCH` | `/api/v1/rooms/{id}/visibility` | Landlord (chủ sở hữu) | Bật/tắt hiển thị tin |
| `PATCH` | `/api/v1/rooms/{id}/occupancy-status` | Landlord (chủ sở hữu) | Chuyển sang `BaoTri` hoặc về `Trong` |
| `POST` | `/api/v1/rooms/{id}/archive` | Landlord (chủ sở hữu) | Lưu trữ phòng |
| `POST` | `/api/v1/rooms/{id}/images` | Landlord (chủ sở hữu) | Tải ảnh phòng |
| `POST` | `/api/v1/admin/rooms/{id}/hide` | Admin | Ẩn tin vi phạm, đặt `DaAnBoiAdmin` |

**Quy tắc quan trọng:**

- `POST /properties/{propertyId}/rooms` tạo phòng ở `occupancyStatus = Trong` và `visibilityStatus = DaAnBoiChuTro` — phòng chưa hiển thị cho tới khi Chủ trọ bật (BP-03).
- `PUT /rooms/{id}` sửa giá chỉ ảnh hưởng hợp đồng lập **sau đó** (BR-12); thao tác này ghi `audit_logs`.
- `PATCH /rooms/{id}/visibility` chỉ nhận `DangHienThi` và `DaAnBoiChuTro`. Phòng đang ở `DaAnBoiAdmin` trả `403` — Chủ trọ không tự bật lại được.
- `PATCH /rooms/{id}/occupancy-status` chỉ nhận hai chuyển tiếp `Trong` → `BaoTri` và `BaoTri` → `Trong`; chuyển tiếp khác trả `409`. Chuyển về `Trong` mà hợp đồng hiện tại chưa ở `DaThanhLy` hoặc `DaHuy` cũng trả `409` (BR-08).
- Không có endpoint `DELETE /rooms/{id}` (BR-09).

---

## 6. Tìm kiếm công khai — BP-04

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/rooms/search` | Công khai | Tìm phòng bằng bộ lọc |
| `GET` | `/api/v1/rooms/{id}/public` | Công khai | Chi tiết phòng và khu trọ cho người tìm phòng |
| `GET` | `/api/v1/amenities` | Công khai | Danh mục tiện ích phục vụ bộ lọc |

**Tham số của `/rooms/search`:** `minPrice`, `maxPrice`, `minArea`, `maxArea`, `city`, `district`, `ward`, `amenityIds` (lặp lại nhiều lần), `minOccupants`, `page`, `pageSize`, `sortBy` (`price` / `area`), `sortDirection`.

Kết quả **chỉ** gồm phòng thỏa mãn đủ điều kiện BR-05. Response không chứa thông tin liên hệ của Chủ trọ (QR-07).

Phase 1 không có endpoint tìm kiếm bằng ngôn ngữ tự nhiên — đó là BP-04 A1 thuộc Phase 2.

---

## 7. Yêu cầu thuê — BP-06

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/rooms/{roomId}/rental-requests` | Tenant | Gửi yêu cầu thuê |
| `GET` | `/api/v1/rental-requests` | Tenant / Landlord | Danh sách theo vai trò người gọi |
| `GET` | `/api/v1/rental-requests/{id}` | Bên liên quan | Chi tiết |
| `POST` | `/api/v1/rental-requests/{id}/approve` | Landlord (chủ sở hữu) | Duyệt |
| `POST` | `/api/v1/rental-requests/{id}/reject` | Landlord (chủ sở hữu) | Từ chối, bắt buộc có `reason` |
| `POST` | `/api/v1/rental-requests/{id}/cancel` | Tenant (người gửi) | Rút yêu cầu |

**`POST /rental-requests/{id}/approve` thực hiện đồng thời trong một transaction:**

1. Chuyển yêu cầu sang `DaDuyet`.
2. Chuyển `rooms.occupancy_status` sang `DangGiuCho` — phòng biến mất khỏi kết quả tìm kiếm.
3. Chuyển **toàn bộ** yêu cầu khác của cùng phòng đang ở `ChoDuyet` sang `TuChoi` với lý do do hệ thống sinh (BR-06), và gửi thông báo cho từng người thuê bị từ chối.

Gửi yêu cầu cho phòng không ở `Trong` trả `409`. Người thuê đã có một yêu cầu `ChoDuyet` cho cùng phòng trả `409` (BR-27).

**Số điện thoại bên còn lại (QR-07):** `GET /rental-requests/{id}` trả kèm số điện thoại của bên còn lại khi yêu cầu ở `DaDuyet`; các trạng thái khác không trả. Sau khi hợp đồng được tạo, số điện thoại hai bên nằm trong `GET /contracts/{id}` cho tới khi hợp đồng ở `DaThanhLy` hoặc `DaHuy`.

---

## 8. Hợp đồng — BP-06

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/contracts` | Landlord (chủ sở hữu) | Lập hợp đồng nháp từ một yêu cầu đã duyệt |
| `GET` | `/api/v1/contracts` | Tenant / Landlord | Danh sách theo vai trò |
| `GET` | `/api/v1/contracts/{id}` | Bên liên quan | Chi tiết, gồm phí dịch vụ và người ở cùng |
| `PUT` | `/api/v1/contracts/{id}` | Landlord (chủ sở hữu) | Sửa khi còn ở `Nhap` |
| `POST` | `/api/v1/contracts/{id}/send` | Landlord (chủ sở hữu) | Gửi cho người thuê xác nhận |
| `POST` | `/api/v1/contracts/{id}/confirm` | Tenant (người đứng tên) | Đồng ý điều khoản |
| `POST` | `/api/v1/contracts/{id}/request-changes` | Tenant (người đứng tên) | Yêu cầu chỉnh sửa, bắt buộc có `reason` |
| `POST` | `/api/v1/contracts/{id}/deposit/confirm` | Landlord (chủ sở hữu) | Xác nhận đã nhận cọc |
| `POST` | `/api/v1/contracts/{id}/cancel` | Landlord / Tenant | Hủy trước ngày bắt đầu hợp đồng, bắt buộc có `reason` |

**`POST /api/v1/contracts`** — body chốt cứng toàn bộ giá tại thời điểm tạo:

```json
{
  "rentalRequestId": 45,
  "rentPrice": 3000000,
  "electricityUnitPrice": 3500,
  "waterUnitPrice": 15000,
  "depositAmount": 3000000,
  "startDate": "2026-10-01",
  "endDate": "2027-09-30",
  "billingCycleDay": 30,
  "paymentDueDays": 7,
  "serviceFees": [
    { "name": "Rac", "amount": 50000 },
    { "name": "Internet", "amount": 100000 }
  ],
  "occupants": [
    { "fullName": "Tran Thi B", "phoneNumber": "09xxxxxxxx" }
  ]
}
```

**Điều kiện chuyển sang `DangHieuLuc` (BR-21):** hệ thống tự kiểm tra sau mỗi lần gọi `/confirm` và `/deposit/confirm`. Khi cả hai đã hoàn tất, hợp đồng chuyển `DangHieuLuc`, phòng chuyển `DangThue`, và cả hai bên nhận thông báo mức Cao.

**Ràng buộc kiểm tra khi tạo:**

- Yêu cầu thuê phải ở `DaDuyet`, trả `409` nếu không. Tạo xong thì yêu cầu thuê chuyển `DaLapHopDong`; hợp đồng bị hủy sau đó không làm đổi trạng thái này.
- Tổng số người ở (người đứng tên + `occupants`) không vượt `maxOccupants` của phòng, nếu vượt trả `422` (BR-11).
- Phòng đã có hợp đồng đang chiếm dụng trả `409` (BR-07).

`POST /deposit/confirm` ghi `audit_logs` theo BR-23.

**`POST /request-changes`** — body gồm `reason` (bắt buộc). Chỉ nhận khi hợp đồng ở `ChoNguoiThueXacNhan`; hợp đồng quay về `Nhap` để Chủ trọ sửa bằng `PUT` rồi gửi lại bằng `/send`. Chủ trọ nhận thông báo mức Cao kèm lý do.

**Hạn giữ chỗ (BP-06 A3):** quá 3 ngày kể từ khi yêu cầu thuê được duyệt mà hợp đồng chưa `DangHieuLuc`, tác vụ định kỳ chuyển hợp đồng (nếu đã lập) sang `DaHuy`, yêu cầu thuê chưa được lập hợp đồng sang `HetHan`, và phòng về `Trong`. Không có endpoint cho việc này.

**Mã VietQR cho tiền cọc:** `GET /contracts/{id}` trả thêm trường `paymentQr` cho Người thuê đứng tên khi hợp đồng ở `ChoNhanCoc` và `depositAmount` > 0, với `amount` bằng `depositAmount` và `transferContent` dạng `SMARTRENT COC<contractId>`. Cấu trúc trường này mô tả ở Mục 9.1.

**`POST /api/v1/contracts/{id}/cancel`** — áp dụng cho hợp đồng ở `Nhap`, `ChoNguoiThueXacNhan`, `ChoNhanCoc`, và cả `DangHieuLuc` khi **chưa tới** `startDate`; gọi trên hợp đồng đã qua `startDate` trả `409` — trường hợp đó phải đi theo luồng thanh lý (BP-10). Hợp đồng chuyển `DaHuy`, phòng trở lại `Trong` và hiển thị lại. Nếu đã có `deposit_received_at`, Chủ trọ hoàn **toàn bộ** cọc: hệ thống không tạo hóa đơn thanh lý và không nhận bất kỳ khoản khấu trừ nào (BR-22, FR-76). Khi đó body phải có thêm `refundedAt` và `refundMethod`; số tiền hoàn do server đặt bằng `depositAmount` và lưu vào `deposit_refunded_*` (FR-86). Việc hoàn cọc ghi `audit_logs` theo BR-23.

**Thông tin hoàn cọc:** `GET /contracts/{id}` trả thêm `depositRefund` (`amount`, `refundedAt`, `method`) khi cọc đã được hoàn, `null` khi chưa.

---

## 9. Hóa đơn và thanh toán — BP-07

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/contracts/{contractId}/invoices` | Landlord (chủ sở hữu) | Chốt chỉ số và tạo hóa đơn nháp |
| `GET` | `/api/v1/contracts/{contractId}/invoices` | Bên liên quan | Danh sách hóa đơn của hợp đồng |
| `GET` | `/api/v1/contracts/{contractId}/meter-readings/latest` | Landlord (chủ sở hữu) | Chỉ số kỳ liền trước, dùng làm chỉ số cũ |
| `GET` | `/api/v1/invoices/{id}` | Bên liên quan | Chi tiết, gồm chỉ số, đơn giá và cách tính |
| `PUT` | `/api/v1/invoices/{id}` | Landlord (chủ sở hữu) | Sửa khi chưa được xác nhận thanh toán |
| `POST` | `/api/v1/invoices/{id}/issue` | Landlord (chủ sở hữu) | Phát hành |
| `POST` | `/api/v1/invoices/{id}/cancel` | Landlord (chủ sở hữu) | Hủy khi chưa thanh toán, bắt buộc có `reason` |
| `POST` | `/api/v1/invoices/{id}/adjustments` | Landlord (chủ sở hữu) | Tạo hóa đơn điều chỉnh cho hóa đơn đã thanh toán |
| `POST` | `/api/v1/invoices/{id}/payment-reports` | Tenant (người đứng tên) | Báo đã thanh toán kèm minh chứng |
| `POST` | `/api/v1/invoices/{id}/payment-reports/{reportId}/confirm` | Landlord (chủ sở hữu) | Xác nhận đã thu |
| `POST` | `/api/v1/invoices/{id}/payment-reports/{reportId}/reject` | Landlord (chủ sở hữu) | Từ chối xác nhận, bắt buộc có `reason` |

**`POST /contracts/{contractId}/invoices`**

```json
{
  "periodStart": "2026-10-01",
  "periodEnd": "2026-10-31",
  "currentElectricityIndex": 1250.0,
  "currentWaterIndex": 84.5,
  "electricityMeterPhotoPath": "...",
  "waterMeterPhotoPath": "...",
  "adjustmentLines": [
    { "category": "DieuChinhKhac", "description": "Tru tien sua vòi nuoc thang truoc", "amount": -50000 }
  ]
}
```

Chỉ số cũ **do hệ thống tự điền** bằng chỉ số mới của kỳ liền trước — client không gửi lên. Đơn giá lấy từ hợp đồng, không lấy từ phòng (BR-13). Server tính `electricityAmount`, `waterAmount`, `rentAmount` và `totalAmount`; client không được gửi các giá trị này.

**Kiểm tra khi tạo:**

- Chỉ số mới nhỏ hơn chỉ số cũ trả `422` (BR-14).
- Kỳ trùng với một hóa đơn `DinhKy` đã có trả `409` (BR-17).
- Kỳ đầu tiên và kỳ cuối không trọn tháng: `rentAmount` tính theo tỷ lệ số ngày thực ở (BR-15).
- Kỳ nằm sau `endDate` vẫn lập được khi hợp đồng chưa có thông báo trả phòng — hợp đồng tiếp tục hiệu lực theo điều khoản đã chốt (FR-88).

**Sửa và điều chỉnh:**

- `PUT /invoices/{id}` chỉ chấp nhận khi hóa đơn ở `Nhap`, `ChuaThanhToan` hoặc `QuaHan`. Hóa đơn ở `DaThanhToan` trả `409` (BR-16). Mỗi lần sửa ghi `audit_logs` với giá trị cũ và mới, và gửi thông báo cho người thuê.
- `POST /invoices/{id}/adjustments` tạo bản ghi mới `type = "DieuChinh"` với `adjustedInvoiceId` trỏ về hóa đơn gốc.

**Luồng thanh toán:**

1. Người thuê gọi `/payment-reports` kèm `proofImagePath` (bắt buộc, BR-06b) khi hóa đơn ở `ChuaThanhToan`, `ThanhToanMotPhan` hoặc `QuaHan` → hóa đơn chuyển `ChoXacNhan`. Trạng thái khác trả `409`.
2. Chủ trọ gọi `/confirm` với `confirmedAmount`; server cộng vào `paidAmount`. Đủ tổng hóa đơn → `DaThanhToan`. Còn thiếu → `QuaHan` nếu đã qua `dueDate`, ngược lại `ThanhToanMotPhan`.
3. Chủ trọ gọi `/reject` → hóa đơn về `QuaHan` nếu đã qua `dueDate`; ngược lại về `ThanhToanMotPhan` nếu `paidAmount` > 0; còn lại về `ChuaThanhToan`.

Chỉ Chủ trọ sở hữu mới xác nhận được thanh toán; mọi vai trò khác trả `403` (BR-06b). Thao tác xác nhận ghi `audit_logs`.

**Quá hạn:** tác vụ định kỳ của hệ thống gắn cờ `QuaHan` cho hóa đơn ở `ChuaThanhToan` hoặc `ThanhToanMotPhan` đã quá `dueDate`, và gửi thông báo mức Cao cho cả hai bên. Không có endpoint cho việc này.

### 9.1 Mã VietQR — BR-26

`GET /invoices/{id}` trả thêm trường `paymentQr` cho Người thuê đứng tên khi hóa đơn ở `ChuaThanhToan`, `QuaHan` hoặc `ThanhToanMotPhan` và số tiền còn phải trả > 0. Quy tắc áp dụng cho mọi loại hóa đơn, kể cả hóa đơn thanh lý có số dư dương.

```json
"paymentQr": {
  "bankBin": "970436",
  "accountNumber": "0123456789",
  "accountName": "NGUYEN VAN A",
  "amount": 2350000,
  "transferContent": "SMARTRENT HD128"
}
```

- `amount` = `totalAmount` − `paidAmount`, **do server tính**; frontend không tự tính lại.
- `transferContent` dạng `SMARTRENT HD<invoiceId>`, chỉ gồm chữ không dấu và chữ số để mọi ứng dụng ngân hàng đọc được.
- `paymentQr` là `null` khi Chủ trọ chưa khai báo tài khoản, khi hóa đơn không ở các trạng thái trên, hoặc khi người gọi không phải Người thuê đứng tên.
- Frontend dựng chuỗi dữ liệu theo chuẩn VietQR và vẽ mã QR ngay trong trình duyệt từ các trường này; thông tin tài khoản không được gửi sang dịch vụ bên ngoài.
- Không có endpoint nhận kết quả chuyển khoản. Người thuê chuyển tiền xong vẫn đi theo luồng thanh toán ở trên: báo đã thanh toán kèm minh chứng → Chủ trọ xác nhận.

---

## 10. Chấm dứt hợp đồng và tất toán cọc — BP-10

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/contracts/{id}/move-out-notice` | Landlord / Tenant | Gửi thông báo trả phòng |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice` | Landlord (chủ sở hữu) | Lập hóa đơn thanh lý |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice/confirm` | Tenant (người đứng tên) | Xác nhận bảng thanh lý |
| `POST` | `/api/v1/contracts/{id}/settlement/complete` | Landlord (chủ sở hữu) | Xác nhận hoàn tất thanh lý |

**`POST /move-out-notice`** — body gồm `expectedMoveOutDate` và `reason`. Hợp đồng chuyển `DangThanhLy`; bên còn lại nhận thông báo mức Cao. Thông báo gửi trước ít hơn 30 ngày vẫn được chấp nhận (FR-87); response ghi rõ số ngày báo trước để hai bên thấy.

**`POST /settlement-invoice`** tạo hóa đơn `type = "ThanhLy"` gồm chỉ số điện nước lần cuối và các dòng chi tiết:

```json
{
  "currentElectricityIndex": 1310.0,
  "currentWaterIndex": 89.0,
  "moveOutDate": "2026-12-15",
  "lines": [
    { "category": "CongNoKyTruoc", "description": "Hoa don thang 11 chua thanh toan", "amount": 500000 },
    { "category": "BoiThuongHuHong", "description": "Vo kinh cua so", "amount": 300000, "evidencePath": "..." }
  ]
}
```

Mỗi khoản khấu trừ **bắt buộc** là một dòng riêng có `description`; gửi một khoản gộp không mô tả trả `422` (BR-22).

**Dòng trừ tiền cọc do server tự thêm:** category `KhauTruTienCoc`, `amount` = −`depositAmount` của hợp đồng (không thêm khi tiền cọc bằng 0). Client gửi dòng `KhauTruTienCoc` trả `422`.

Tổng các dòng `PhiPhat` không được vượt `depositAmount`, vượt trả `422` (FR-87).

`totalAmount` âm nghĩa là Chủ trọ phải hoàn lại phần cọc dư. `totalAmount` dương thì Người thuê thấy mã VietQR trên hóa đơn thanh lý theo Mục 9.1.

**`POST /settlement/complete`** chỉ thực hiện được khi người thuê đã xác nhận bảng thanh lý, và:

- `totalAmount` dương: hóa đơn thanh lý đã `DaThanhToan`;
- `totalAmount` âm: body có `refundedAt` và `refundMethod`; số tiền hoàn do server đặt bằng −`totalAmount` và lưu vào `deposit_refunded_*` (FR-86).

Kết quả: hợp đồng chuyển `DaThanhLy`, phòng chuyển `BaoTri` hoặc `Trong` theo tham số `roomNextStatus`, hai bên nhận thông báo mức Cao. Thao tác này ghi `audit_logs`, gồm cả thông tin hoàn cọc.

---

## 11. Thông báo

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/notifications` | Đã đăng nhập | Danh sách thông báo của chính mình, lọc theo `isRead` |
| `GET` | `/api/v1/notifications/unread-count` | Đã đăng nhập | Số thông báo chưa đọc |
| `POST` | `/api/v1/notifications/{id}/read` | Người nhận | Đánh dấu đã đọc |
| `POST` | `/api/v1/notifications/read-all` | Đã đăng nhập | Đánh dấu tất cả đã đọc |

Thông báo chỉ được sinh bởi backend khi sự kiện nghiệp vụ xảy ra. Không có endpoint tạo thông báo thủ công.

---

## 12. Dashboard

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/dashboard/admin` | Admin | Tổng người dùng, chủ trọ, người thuê, khu trọ, phòng, hồ sơ chờ duyệt |
| `GET` | `/api/v1/dashboard/landlord` | Landlord | Số phòng trống/đang thuê, hóa đơn chưa thu, doanh thu đã xác nhận theo tháng |
| `GET` | `/api/v1/dashboard/tenant` | Tenant | Hợp đồng hiện tại, hóa đơn chưa thanh toán, tổng đã thanh toán |

Doanh thu chỉ tính từ các khoản đã được Chủ trọ xác nhận thu.

---

## 13. Nhật ký hệ thống

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/admin/audit-logs` | Admin | Tra cứu nhật ký, lọc theo `entityType`, `entityId`, `actorUserId`, khoảng thời gian |

Chỉ có thao tác đọc. Không có endpoint tạo, sửa hay xóa nhật ký, kể cả cho Admin (QR-03).

---

## 14. Tải lên file

Toàn bộ ảnh của hệ thống — giấy tờ nhân thân, ảnh khu trọ và phòng, ảnh đồng hồ điện nước, minh chứng thanh toán, ảnh hư hỏng khi thanh lý — đi qua **một endpoint dùng chung**.

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/files` | Đã đăng nhập | Tải lên một file, trả về `path` để gắn vào request nghiệp vụ sau đó |

**Request** dạng `multipart/form-data` với hai phần: `file` và `purpose`.

`purpose` nhận một trong các giá trị: `GiayToNhanThan` · `AnhKhuTro` · `AnhPhong` · `AnhDongHo` · `MinhChungThanhToan` · `AnhHuHong`.

**Response**

```json
{
  "path": "public-media/AnhPhong/12/2026/09/3f2a...c1.jpg",
  "url": "https://<project>.supabase.co/storage/v1/object/public/public-media/AnhPhong/12/2026/09/3f2a...c1.jpg",
  "purpose": "AnhPhong"
}
```

- `path` là giá trị client gửi lại trong request nghiệp vụ tiếp theo (các trường tên `...Path`). Dạng: `<bucket>/<purpose>/<id người tải>/<năm>/<tháng>/<tên do server sinh><phần mở rộng>`.
- `url` chỉ dùng để xem trước: URL vĩnh viễn với ảnh công khai, URL có chữ ký hết hạn sau 1 giờ với file riêng tư. Với `GiayToNhanThan`, `url` là `null` — ảnh giấy tờ chỉ Admin xem được (FR-10); giao diện xem trước bằng file đang có trong trình duyệt.

**Quy tắc bắt buộc:**

- File lưu trên **Supabase Storage**, chia hai bucket:

| Bucket | Công khai | Giới hạn | Định dạng | Dùng cho |
|---|---|---|---|---|
| `public-media` | Có | 5 MB | jpeg, png, webp | Ảnh khu trọ, ảnh phòng |
| `private-documents` | Không | 10 MB | jpeg, png, webp, pdf | Giấy tờ nhân thân, ảnh đồng hồ, minh chứng thanh toán, ảnh hư hỏng |

- `purpose` quyết định bucket: `AnhKhuTro` và `AnhPhong` vào `public-media`, bốn giá trị còn lại vào `private-documents`.
- File trong bucket riêng tư **chỉ** truy cập được qua URL có chữ ký và có hạn, sinh ra tại thời điểm người có quyền yêu cầu xem. Không bao giờ trả URL công khai vĩnh viễn cho các loại này.
- `purpose` quyết định bucket và quyền. Endpoint kiểm tra vai trò người gọi có được phép tải loại file đó lên hay không: `GiayToNhanThan` và `MinhChungThanhToan` chỉ Người thuê; `AnhKhuTro`, `AnhPhong`, `AnhDongHo`, `AnhHuHong` chỉ Chủ trọ.
- Server xác định định dạng từ nội dung file (chữ ký ở các byte đầu) và kiểm tra dung lượng trước khi lưu; không tin phần mở rộng hay `Content-Type` do client khai. Phần mở rộng lưu trên Storage do server đặt theo định dạng đã nhận diện.
- Khi nhận `path` trong request nghiệp vụ, server kiểm tra đường dẫn đúng bucket, đúng thư mục của `purpose` tương ứng, và nằm trong thư mục của chính người gọi; sai thì trả `422`. Không nhận đường dẫn hay URL tùy ý từ client.
- Giới hạn 20 lần tải mỗi giờ cho mỗi tài khoản (Thiết kế An toàn mục 6).
