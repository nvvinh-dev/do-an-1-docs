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
| **Ngày giờ** | Thời điểm: ISO 8601, múi giờ UTC. Trường chỉ có ngày (`startDate`, `dueDate`...): `YYYY-MM-DD` theo lịch Việt Nam (Kiến trúc mục 7.1) |
| **Tiền** | Số, không định dạng, không kèm đơn vị. Số tiền do server tính được làm tròn đến đồng |

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
- Khi người gọi không có quyền với một tài nguyên tồn tại, trả `404` thay vì `403` đối với các tài nguyên có thể bị dò id — khu trọ, phòng, hợp đồng, hóa đơn, hồ sơ Chủ trọ.
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

**`POST /api/v1/auth/forgot-password`** luôn trả `200` với cùng một thông báo và trả về ngay, dù email có tồn tại hay không. Email được gửi nền; lỗi gửi chỉ ghi log, không làm request thất bại.

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

Ba đường dẫn là giá trị `path` nhận được từ `POST /files` với `purpose = GiayToNhanThan`, phải do chính người nộp tải lên và file phải tồn tại (Mục 14); sai thì trả `422`. Tài khoản chưa có số điện thoại trả `422`. Đã có hồ sơ ở `ChoDuyet` trả `409` — kể cả khi hai lượt nộp gửi cùng lúc, nhờ unique index ở database (FR-07). Tài khoản còn hợp đồng chưa kết thúc (khác `DaThanhLy`, `DaHuy`) hoặc còn yêu cầu thuê ở `ChoDuyet`, `DaDuyet` trả `409` (BR-01, FR-97); `/approve` kiểm tra lại điều kiện này.

**`POST /approve`** — Admin gọi xác minh số điện thoại của người nộp trước khi duyệt, rồi gửi lại đúng số đã gọi:

```json
{ "verifiedPhoneNumber": "0901234567" }
```

Số này khác số hiện tại của người nộp — tức người nộp đã đổi số trong lúc Admin xác minh — thì trả `409`, Admin gọi xác minh lại số mới. Trùng thì duyệt: tài khoản được cấp vai trò `Landlord` thay cho `Tenant`, số điện thoại được ghi nhận là đã xác thực (BR-01), và số đã xác minh được ghi vào `audit_logs`. Người nộp phải đăng nhập lại để token mang vai trò mới. Người nộp đã là Chủ trọ (hồ sơ cũ còn sót ở `ChoDuyet`) thì trả `409`; Admin từ chối hồ sơ đó.

Duyệt và từ chối đều ghi `audit_logs` theo BR-23. Người nộp nhận thông báo mức Cao.

---

## 4. Quản lý tài khoản — BP-01

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/admin/users` | Admin | Danh sách tài khoản, lọc theo vai trò và trạng thái khóa |
| `POST` | `/api/v1/admin/users/{id}/lock` | Admin | Khóa tài khoản, bắt buộc có `reason` |
| `POST` | `/api/v1/admin/users/{id}/unlock` | Admin | Mở khóa |

Cả hai thao tác ghi `audit_logs` và gửi thông báo mức Cao cho người bị xử lý.

`GET /admin/users` trả thêm `activeContractCount` — số hợp đồng chưa kết thúc — với tài khoản Chủ trọ, để giao diện cảnh báo trước khi khóa (FR-09). **Giới hạn được chấp nhận ở Phase 1:** Chủ trọ bị khóa không đăng nhập được, nên các hóa đơn của người thuê dừng ở `ChoXacNhan` cho tới khi được mở khóa, còn hợp đồng ở `ChoNhanCoc` không được xác nhận cọc và có thể hết hạn giữ chỗ (BP-01 A2).

---

## 5. Khu trọ và phòng trọ — BP-02, BP-03

### 5.1 Khu trọ

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/properties` | Landlord | Tạo khu trọ |
| `GET` | `/api/v1/properties` | Landlord | Danh sách khu trọ của chính mình; mặc định ẩn khu đã lưu trữ, `includeArchived=true` để xem cả |
| `GET` | `/api/v1/properties/{id}` | Landlord (chủ sở hữu) | Chi tiết |
| `PUT` | `/api/v1/properties/{id}` | Landlord (chủ sở hữu) | Cập nhật |
| `POST` | `/api/v1/properties/{id}/archive` | Landlord (chủ sở hữu) | Lưu trữ khu trọ |

**`POST /api/v1/properties`** và **`PUT /api/v1/properties/{id}`** — cùng một body:

```json
{
  "name": "Nhà trọ Hoa Mai",
  "address": "12 đường số 5, gần ĐH Công nghệ Thông tin",
  "city": "Thành phố Hồ Chí Minh",
  "ward": "<tên phường/xã trong danh mục>",
  "description": "Khu yên tĩnh, có bảo vệ",
  "amenityIds": [1, 3],
  "imagePaths": ["public-media/AnhKhuTro/12/2026/10/3f2a...c1.jpg"]
}
```

- `name`, `address`, `city`, `ward` bắt buộc. Cặp (`city`, `ward`) phải có trong danh mục `GET /locations` (Mục 6), sai trả `422`. Tên quận cũ, nếu muốn cho người đọc dễ nhận biết, ghi trong `address`.
- `amenityIds` chỉ nhận tiện ích có `scope = KhuTro`, sai trả `422`.
- `imagePaths` tối đa 10 đường dẫn, mỗi đường dẫn là file `AnhKhuTro` do chính Chủ trọ tải lên (Mục 14), sai trả `422`. Thứ tự trong mảng là thứ tự hiển thị; ảnh đầu tiên là ảnh đại diện.
- `PUT` thay toàn bộ danh sách tiện ích và ảnh bằng danh sách mới. Ảnh bị bỏ khỏi danh sách không bị xóa khỏi Storage ở Phase 1.
- Khu trọ ở `LuuTru` chỉ còn xem được: `PUT` và thêm phòng mới đều trả `409`.

**Response** — `POST` trả `201`; `GET /properties/{id}` và `PUT` trả `200`:

```json
{
  "id": 3,
  "name": "Nhà trọ Hoa Mai",
  "address": "12 đường số 5, gần ĐH Công nghệ Thông tin",
  "city": "Thành phố Hồ Chí Minh",
  "ward": "<tên phường/xã>",
  "description": "Khu yên tĩnh, có bảo vệ",
  "status": "DangKhaiThac",
  "amenities": [{ "id": 1, "name": "<tên tiện ích>" }],
  "images": [{ "path": "public-media/AnhKhuTro/12/2026/10/3f2a...c1.jpg", "url": "https://..." }],
  "roomCounts": { "total": 8, "trong": 2, "dangGiuCho": 1, "dangThue": 4, "baoTri": 1 }
}
```

`roomCounts` không tính phòng đã lưu trữ. `GET /properties` trả danh sách gồm `id`, `name`, `address`, `city`, `ward`, `status`, `coverImageUrl`, `roomCounts`, không phân trang — mỗi Chủ trọ quản lý dưới 50 phòng (AS-04).

**`POST /properties/{id}/archive`** — không có body, trả `204`. Trả `409` khi còn phòng ở `DangGiuCho` hoặc `DangThue` (BR-10). Lưu trữ khu trọ thì mọi phòng của khu chuyển `LuuTru` theo, và các yêu cầu thuê `ChoDuyet` của các phòng đó tự chuyển `TuChoi` với lý do do hệ thống sinh — tất cả trong cùng transaction. Không có endpoint `DELETE` (BR-09).

### 5.2 Phòng trọ

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/properties/{propertyId}/rooms` | Landlord (chủ sở hữu) | Thêm phòng |
| `GET` | `/api/v1/properties/{propertyId}/rooms` | Landlord (chủ sở hữu) | Danh sách phòng của khu trọ; mặc định ẩn phòng đã lưu trữ, `includeArchived=true` để xem cả |
| `GET` | `/api/v1/rooms/{id}` | Landlord (chủ sở hữu) | Chi tiết phòng ở góc nhìn quản lý |
| `PUT` | `/api/v1/rooms/{id}` | Landlord (chủ sở hữu) | Cập nhật thông tin, giá, phí dịch vụ, tiện ích, ảnh |
| `PATCH` | `/api/v1/rooms/{id}/visibility` | Landlord (chủ sở hữu) | Bật/tắt hiển thị tin |
| `PATCH` | `/api/v1/rooms/{id}/occupancy-status` | Landlord (chủ sở hữu) | Chuyển sang `BaoTri` hoặc về `Trong` |
| `POST` | `/api/v1/rooms/{id}/archive` | Landlord (chủ sở hữu) | Lưu trữ phòng |

**`POST /api/v1/properties/{propertyId}/rooms`** và **`PUT /api/v1/rooms/{id}`** — cùng một body:

```json
{
  "code": "P101",
  "area": 20.5,
  "maxOccupants": 2,
  "rentPrice": 3000000,
  "electricityUnitPrice": 3500,
  "waterUnitPrice": 15000,
  "description": "Có gác lửng, cửa sổ thoáng",
  "serviceFees": [
    { "name": "Rác", "amount": 20000 },
    { "name": "Wifi", "amount": 100000 }
  ],
  "amenityIds": [7, 9],
  "imagePaths": ["public-media/AnhPhong/12/2026/10/8b1c...d4.jpg"]
}
```

- `code` bắt buộc, không trùng với phòng khác trong cùng khu trọ, trùng trả `409`.
- `area` > 0, `maxOccupants` ≥ 1, `rentPrice` > 0; `electricityUnitPrice` và `waterUnitPrice` ≥ 0. Sai trả `422`.
- `serviceFees` tối đa 10 khoản; mỗi khoản có `name` không trùng trong phòng và `amount` ≥ 0.
- `amenityIds` chỉ nhận tiện ích có `scope = Phong`. `imagePaths` theo cùng quy tắc với khu trọ, với file `AnhPhong`. Sai trả `422`.
- `PUT` thay toàn bộ phí dịch vụ, tiện ích và ảnh bằng danh sách mới; không đổi được khu trọ của phòng. Phòng ở `LuuTru` trả `409`.
- Phòng đang `DangHienThi` mà `PUT` bỏ hết ảnh thì trả `422` — Chủ trọ tắt hiển thị trước.
- Đổi giá thuê, đơn giá điện, đơn giá nước hoặc danh sách phí dịch vụ thì ghi `audit_logs` mã `SuaGiaPhong` với giá trị cũ và mới (BR-23). Thay đổi chỉ áp dụng cho hợp đồng lập **sau đó** (BR-12).

**Response** — `POST` trả `201`; `GET /rooms/{id}` và `PUT` trả `200`:

```json
{
  "id": 21,
  "propertyId": 3,
  "propertyName": "Nhà trọ Hoa Mai",
  "code": "P101",
  "area": 20.5,
  "maxOccupants": 2,
  "rentPrice": 3000000,
  "electricityUnitPrice": 3500,
  "waterUnitPrice": 15000,
  "description": "Có gác lửng, cửa sổ thoáng",
  "occupancyStatus": "Trong",
  "visibilityStatus": "DangHienThi",
  "isListed": true,
  "serviceFees": [{ "name": "Rác", "amount": 20000 }],
  "amenities": [{ "id": 7, "name": "<tên tiện ích>" }],
  "images": [{ "path": "public-media/AnhPhong/12/2026/10/8b1c...d4.jpg", "url": "https://..." }],
  "currentContractId": null
}
```

- `isListed` cho biết phòng có đang xuất hiện trong tìm kiếm không — đủ cả bốn điều kiện BR-05 — để Chủ trọ hiểu vì sao một phòng đã bật hiển thị mà vẫn không ai thấy.
- `currentContractId` là hợp đồng chưa kết thúc gần nhất của phòng (từ `Nhap` tới `DangThanhLy`), `null` nếu không có.
- `GET /properties/{propertyId}/rooms` trả danh sách gồm `id`, `code`, `area`, `maxOccupants`, `rentPrice`, `occupancyStatus`, `visibilityStatus`, `isListed`, `coverImageUrl`, không phân trang (AS-04).

**`PATCH /rooms/{id}/visibility`** — body `{ "visibilityStatus": "DangHienThi" }`, trả `204`. Chỉ nhận `DangHienThi` và `DaAnBoiChuTro`. Phòng đang ở `DaAnBoiAdmin` trả `409` — Chủ trọ không tự bật lại được. Phòng ở `LuuTru` trả `409`. Bật `DangHienThi` khi phòng chưa có ảnh nào trả `422`.

**`PATCH /rooms/{id}/occupancy-status`** — body `{ "occupancyStatus": "BaoTri" }`, trả `204`. Chỉ nhận hai chuyển tiếp `Trong` → `BaoTri` và `BaoTri` → `Trong`; chuyển tiếp khác trả `409`. Chuyển về `Trong` mà hợp đồng hiện tại chưa ở `DaThanhLy` hoặc `DaHuy` cũng trả `409` (BR-08).

**`POST /rooms/{id}/archive`** — không có body, trả `204`. Chỉ nhận phòng ở `Trong` hoặc `BaoTri`, trạng thái khác trả `409`. Các yêu cầu thuê `ChoDuyet` của phòng tự chuyển `TuChoi` với lý do do hệ thống sinh. Lưu trữ là vĩnh viễn — không có thao tác bỏ lưu trữ.

**Quy tắc chung:**

- `POST /properties/{propertyId}/rooms` tạo phòng ở `occupancyStatus = Trong` và `visibilityStatus = DaAnBoiChuTro` — phòng chưa hiển thị cho tới khi Chủ trọ bật (BP-03).
- Không có endpoint `DELETE /rooms/{id}` (BR-09).
- Ẩn tin vi phạm theo từng phòng (`DaAnBoiAdmin`) thuộc Phase 2, làm cùng khiếu nại BP-13. Phase 1 Admin xử lý vi phạm bằng cách khóa tài khoản Chủ trọ — toàn bộ phòng của Chủ trọ đó rời khỏi kết quả tìm kiếm theo BR-05.

---

## 6. Tìm kiếm công khai — BP-04

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `GET` | `/api/v1/rooms/search` | Công khai | Tìm phòng bằng bộ lọc |
| `GET` | `/api/v1/rooms/{id}/public` | Công khai | Chi tiết phòng và khu trọ cho người tìm phòng |
| `GET` | `/api/v1/amenities` | Công khai | Danh mục tiện ích, gồm `id`, `name`, `scope` |
| `GET` | `/api/v1/locations` | Công khai | Danh mục tỉnh/thành và phường/xã |

**`GET /api/v1/locations`** trả danh mục đơn vị hành chính 2 cấp hiện hành (từ 01/07/2025 không còn cấp quận/huyện), dạng `[{ "city": "...", "wards": ["...", "..."] }]`. Danh mục là file JSON tĩnh trong backend, lấy từ danh mục đơn vị hành chính chính thức sau sắp xếp năm 2025; frontend dùng cho ô chọn địa chỉ và bộ lọc, backend dùng để kiểm tra địa chỉ khu trọ (Mục 5.1).

**Tham số của `/rooms/search`:** `minPrice`, `maxPrice`, `minArea`, `maxArea`, `city`, `ward`, `amenityIds` (lặp lại nhiều lần), `minOccupants`, `page`, `pageSize`, `sortBy` (`price` / `area`), `sortDirection`. Với `amenityIds`, phòng phải có đủ mọi tiện ích được chọn, tính cả tiện ích của phòng và của khu trọ chứa phòng.

Kết quả **chỉ** gồm phòng thỏa mãn đủ điều kiện BR-05. Response không chứa thông tin liên hệ của Chủ trọ (QR-07).

`GET /rooms/{id}/public` cũng chỉ trả phòng đủ điều kiện BR-05, ngoài ra trả `404`. Người thuê xem lại phòng mình đã gửi yêu cầu hoặc đang thuê qua chi tiết yêu cầu thuê và hợp đồng.

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

**`POST /rental-requests/{id}/approve`** chỉ nhận khi yêu cầu ở `ChoDuyet`, chưa quá 168 giờ kể từ lúc gửi — kể cả khi tác vụ định kỳ chưa kịp chuyển nó sang `HetHan` — và phòng còn ở `Trong`; ngược lại trả `409`. Khi duyệt, hệ thống **thực hiện đồng thời trong một transaction:**

1. Chuyển yêu cầu sang `DaDuyet`.
2. Chuyển `rooms.occupancy_status` sang `DangGiuCho` — phòng biến mất khỏi kết quả tìm kiếm.
3. Chuyển **toàn bộ** yêu cầu khác của cùng phòng đang ở `ChoDuyet` sang `TuChoi` với lý do do hệ thống sinh (BR-06), và gửi thông báo cho từng người thuê bị từ chối.

Gửi yêu cầu cho phòng không đủ điều kiện BR-05 trả `404`, giống chi tiết công khai — không để lộ phòng đang ẩn qua việc dò id (FR-26). Người thuê đã có một yêu cầu `ChoDuyet` cho cùng phòng trả `409` (BR-27).

**`POST /rental-requests/{id}/cancel`** nhận khi yêu cầu ở `ChoDuyet`, hoặc ở `DaDuyet` mà chưa lập hợp đồng; trạng thái khác trả `409`. Rút yêu cầu `DaDuyet` thì phòng về `Trong` trong cùng transaction và Chủ trọ nhận thông báo mức Cao (FR-27).

**Số điện thoại bên còn lại (QR-07):** `GET /rental-requests/{id}` trả kèm số điện thoại của bên còn lại khi yêu cầu ở `DaDuyet`; các trạng thái khác không trả. Sau khi hợp đồng được tạo, số điện thoại hai bên nằm trong `GET /contracts/{id}` cho tới khi hợp đồng kết thúc hẳn: `DaThanhLy` mà hóa đơn thanh lý không còn nợ, hoặc `DaHuy` mà không còn chờ hoàn cọc — để hai bên còn liên lạc được khi còn nghĩa vụ tiền.

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
| `POST` | `/api/v1/contracts/{id}/deposit/refund` | Landlord (chủ sở hữu) | Ghi nhận đã hoàn cọc cho hợp đồng đã hủy |
| `GET` | `/api/v1/rooms/{id}/meter-readings/latest` | Landlord (chủ sở hữu) | Chỉ số điện nước cuối cùng đã ghi nhận của phòng, dùng điền sẵn chỉ số đầu khi lập hợp đồng |
| `PATCH` | `/api/v1/contracts/{id}/initial-meter-readings` | Landlord (chủ sở hữu) | Sửa chỉ số đầu khi hợp đồng đã hiệu lực và chưa có hóa đơn |

**`POST /api/v1/contracts`** — body chốt cứng toàn bộ giá tại thời điểm tạo:

```json
{
  "rentalRequestId": 45,
  "rentPrice": 3000000,
  "electricityUnitPrice": 3500,
  "waterUnitPrice": 15000,
  "depositAmount": 3000000,
  "initialElectricityIndex": 1180.0,
  "initialWaterIndex": 76.0,
  "startDate": "2026-10-01",
  "endDate": "2027-09-30",
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

**Điều kiện chuyển sang `DangHieuLuc` (BR-21):** các bước đi tuần tự.

- `/confirm` chỉ nhận ở `ChoNguoiThueXacNhan`. Tiền cọc > 0 thì hợp đồng sang `ChoNhanCoc`; tiền cọc = 0 thì sang thẳng `DangHieuLuc`.
- `/deposit/confirm` chỉ nhận ở `ChoNhanCoc`, trạng thái khác trả `409`. Body gồm `receivedAt` — thời điểm nhận cọc thực tế, không sau thời điểm hiện tại, sai trả `422` — và `method`.

Khi hợp đồng sang `DangHieuLuc`, phòng chuyển `DangThue` và cả hai bên nhận thông báo mức Cao.

**Ràng buộc kiểm tra khi tạo:**

- Yêu cầu thuê phải ở `DaDuyet`, trả `409` nếu không. Tạo xong thì yêu cầu thuê chuyển `DaLapHopDong`; hợp đồng bị hủy sau đó không làm đổi trạng thái này.
- Tổng số người ở (người đứng tên + `occupants`) không vượt `maxOccupants` của phòng, nếu vượt trả `422` (BR-11).
- Phòng đã có hợp đồng đang chiếm dụng trả `409` (BR-07).
- `startDate` không được trước ngày lập hợp đồng (giờ Việt Nam), `endDate` phải sau `startDate`; sai trả `422`.

`POST /deposit/confirm` ghi `audit_logs` theo BR-23.

**Chỉ số đầu (BR-14, FR-90):** `initialElectricityIndex` và `initialWaterIndex` bắt buộc khi tạo hợp đồng, là chỉ số cũ của hóa đơn đầu tiên. Giá trị điền sẵn lấy từ `GET /rooms/{id}/meter-readings/latest`: chỉ số mới của hóa đơn chưa hủy gần nhất thuộc các hợp đồng của phòng, hoặc chỉ số đầu của hợp đồng gần nhất nếu hợp đồng đó chưa có hóa đơn; `null` khi phòng chưa từng có hợp đồng. `PATCH /contracts/{id}/initial-meter-readings` chỉ nhận khi hợp đồng ở `DangHieuLuc` và chưa có hóa đơn nào khác `DaHuy`, ngược lại trả `409`; mỗi lần sửa ghi `audit_logs` và gửi thông báo mức Cao cho người thuê.

**`POST /request-changes`** — body gồm `reason` (bắt buộc). Chỉ nhận khi hợp đồng ở `ChoNguoiThueXacNhan`; hợp đồng quay về `Nhap` để Chủ trọ sửa bằng `PUT` rồi gửi lại bằng `/send`. Chủ trọ nhận thông báo mức Cao kèm lý do.

**Hạn giữ chỗ (BP-06 A3):** quá 72 giờ (3 ngày) kể từ khi yêu cầu thuê được duyệt mà hợp đồng chưa `DangHieuLuc`, tác vụ định kỳ chuyển hợp đồng (nếu đã lập) sang `DaHuy`, yêu cầu thuê chưa được lập hợp đồng sang `HetHan`, và phòng về `Trong`. Không có endpoint cho việc này. Đã quá 72 giờ thì `POST /contracts`, `PUT`, `/send`, `/confirm`, `/deposit/confirm` trên hợp đồng chưa hiệu lực đều trả `409`, kể cả khi tác vụ chưa chạy.

**Mã VietQR cho tiền cọc:** `GET /contracts/{id}` trả thêm trường `paymentQr` cho Người thuê đứng tên khi hợp đồng ở `ChoNhanCoc` và `depositAmount` > 0, với `amount` bằng `depositAmount` và `transferContent` dạng `SMARTRENT COC<contractId>`. Cấu trúc trường này mô tả ở Mục 9.1.

**`POST /api/v1/contracts/{id}/cancel`** — body gồm `reason`. Áp dụng cho hợp đồng ở `Nhap`, `ChoNguoiThueXacNhan`, `ChoNhanCoc`, và cả `DangHieuLuc` khi **chưa tới** `startDate`; gọi trên hợp đồng đã qua `startDate` trả `409` — trường hợp đó phải đi theo luồng thanh lý (BP-10). Hợp đồng chuyển `DaHuy`, ghi bên hủy vào `cancelled_by_user_id`, phòng trở lại `Trong` và hiển thị lại, bên còn lại nhận thông báo mức Cao. Hệ thống không tạo hóa đơn thanh lý (BR-22, FR-76).

**`POST /api/v1/contracts/{id}/deposit/refund`** — chỉ nhận khi hợp đồng ở `DaHuy`, đã có `deposit_received_at` và chưa ghi nhận hoàn cọc; ngược lại trả `409`. Body gồm `refundedAt`, `refundMethod`, và hai trường chỉ dùng khi **Người thuê** là bên hủy: `amount` (từ 0 tới `depositAmount`, sai trả `422`) và `note` (bắt buộc khi `amount` nhỏ hơn `depositAmount`). Khi Chủ trọ là bên hủy, server đặt số hoàn bằng `depositAmount` và bỏ qua `amount`. Lưu vào `deposit_refund*`, ghi `audit_logs` theo BR-23, người thuê nhận thông báo mức Cao (FR-86).

**Thông tin hoàn cọc:** `GET /contracts/{id}` trả thêm `depositRefund` (`amount`, `refundedAt`, `method`, `note`) khi cọc đã được hoàn, `null` khi chưa; và `depositRefundPending` = `true` khi hợp đồng ở `DaHuy`, đã nhận cọc mà chưa ghi nhận hoàn.

---

## 9. Hóa đơn và thanh toán — BP-07

| Method | Endpoint | Quyền | Mô tả |
|---|---|---|---|
| `POST` | `/api/v1/contracts/{contractId}/invoices` | Landlord (chủ sở hữu) | Chốt chỉ số và tạo hóa đơn nháp |
| `GET` | `/api/v1/contracts/{contractId}/invoices` | Bên liên quan | Danh sách hóa đơn của hợp đồng |
| `GET` | `/api/v1/contracts/{contractId}/meter-readings/latest` | Landlord (chủ sở hữu) | Chỉ số cũ của kỳ kế tiếp: chỉ số mới của hóa đơn chưa hủy gần nhất, hoặc chỉ số đầu của hợp đồng nếu chưa có hóa đơn |
| `GET` | `/api/v1/invoices/{id}` | Bên liên quan | Chi tiết, gồm chỉ số, đơn giá và cách tính |
| `PUT` | `/api/v1/invoices/{id}` | Landlord (chủ sở hữu) | Sửa hóa đơn mới nhất của hợp đồng khi chưa thanh toán |
| `POST` | `/api/v1/invoices/{id}/issue` | Landlord (chủ sở hữu) | Phát hành |
| `POST` | `/api/v1/invoices/{id}/cancel` | Landlord (chủ sở hữu) | Hủy hóa đơn mới nhất của hợp đồng khi chưa thanh toán, bắt buộc có `reason` |
| `POST` | `/api/v1/invoices/{id}/payment-reports` | Tenant (người đứng tên) | Báo đã thanh toán kèm minh chứng |
| `POST` | `/api/v1/invoices/{id}/payment-reports/{reportId}/confirm` | Landlord (chủ sở hữu) | Xác nhận đã thu |
| `POST` | `/api/v1/invoices/{id}/payment-reports/{reportId}/reject` | Landlord (chủ sở hữu) | Từ chối xác nhận, bắt buộc có `reason` |

**`POST /contracts/{contractId}/invoices`**

```json
{
  "currentElectricityIndex": 1250.0,
  "currentWaterIndex": 84.5,
  "electricityMeterPhotoPath": "...",
  "waterMeterPhotoPath": "...",
  "adjustmentLines": [
    { "category": "DieuChinhKhac", "description": "Thang 10 ghi du 20 so dien", "amount": -70000, "relatedInvoiceId": 128 }
  ]
}
```

Chỉ số cũ **do hệ thống tự điền** bằng chỉ số mới của kỳ liền trước, kỳ đầu tiên lấy chỉ số đầu ghi trong hợp đồng — client không gửi lên. Đơn giá lấy từ hợp đồng, không lấy từ phòng (BR-13). Server tính `electricityAmount`, `waterAmount`, `rentAmount`, `serviceFeeAmount` và `totalAmount`; client không được gửi các giá trị này.

**Kỳ hóa đơn do server xác định (BR-15, BR-17):** endpoint luôn tạo hóa đơn cho **kỳ kế tiếp** của hợp đồng — tháng liền sau kỳ chưa hủy gần nhất, hoặc từ `startDate` tới cuối tháng đó nếu hợp đồng chưa có hóa đơn. Client không gửi `periodStart`, `periodEnd`; response trả về hai trường này.

**Kiểm tra khi tạo:**

- Chỉ số mới nhỏ hơn chỉ số cũ trả `422` (BR-14).
- Hợp đồng không ở `DangHieuLuc`, `SapHetHan`, `DangThanhLy`, hoặc chưa tới `startDate`, trả `409` (FR-91).
- Kỳ kế tiếp chưa tới — hôm nay trước ngày 25 của tháng đó — trả `409` (FR-91).
- Hợp đồng ở `DangThanhLy` mà kỳ kế tiếp là tháng chứa `expectedMoveOutDate` trả `409` — tháng đó thuộc hóa đơn thanh lý.
- Unique index BR-17 là lớp chặn cuối khi hai request tạo cùng một kỳ chạy song song; vi phạm trả `409`.
- `adjustmentLines` chỉ nhận category `DieuChinhKhac`, loại khác trả `422`. `totalAmount` của hóa đơn định kỳ không được âm, âm trả `422` — khoản giảm lớn hơn tiền tháng thì chia sang các kỳ sau.
- Kỳ đầu tiên và kỳ cuối không trọn tháng: `rentAmount` và `serviceFeeAmount` = giá × số ngày ở ÷ số ngày của tháng, tính cả ngày vào ở và ngày trả phòng, làm tròn đến đồng (BR-15). Ví dụ vào ở 15/10, giá 3.000.000 → 3.000.000 × 17 ÷ 31 = 1.645.161.
- Kỳ nằm sau `endDate` vẫn lập được — hợp đồng tiếp tục hiệu lực theo điều khoản đã chốt (FR-88); khi đã có thông báo trả phòng thì áp quy tắc tháng trả phòng ở trên.

**Sửa và điều chỉnh:**

- `PUT /invoices/{id}` và `POST /invoices/{id}/cancel` chỉ chấp nhận với hóa đơn định kỳ **mới nhất** chưa hủy của hợp đồng; hóa đơn cũ hơn trả `409`. `PUT` nhận khi hóa đơn ở `Nhap`, `ChuaThanhToan`, hoặc `QuaHan` mà chưa thu đồng nào (`paidAmount` = 0); `cancel` nhận khi hóa đơn ở `ChuaThanhToan`. Hóa đơn ở `DaThanhToan` trả `409` (BR-16). Mỗi lần sửa ghi `audit_logs` với giá trị cũ và mới; hóa đơn đã phát hành thì gửi thêm thông báo cho người thuê — hóa đơn `Nhap` người thuê chưa thấy nên không thông báo.
- Hóa đơn cũ hơn hoặc đã thanh toán có sai sót: Chủ trọ thêm một dòng `DieuChinhKhac` có `relatedInvoiceId` vào `adjustmentLines` của hóa đơn kỳ kế tiếp, hoặc vào `lines` của hóa đơn thanh lý. `relatedInvoiceId` phải thuộc cùng hợp đồng, sai thì trả `422`.

**Người thuê không thấy hóa đơn Nháp:** hóa đơn ở `Nhap` không có trong danh sách của người thuê, và `GET /invoices/{id}` trả `404` với người thuê. Người thuê thấy hóa đơn từ khi phát hành.

**Luồng thanh toán:**

1. Người thuê gọi `/payment-reports` kèm `proofImagePath` (bắt buộc, BR-06b) khi hóa đơn ở `ChuaThanhToan`, `ThanhToanMotPhan` hoặc `QuaHan` → hóa đơn chuyển `ChoXacNhan`. Trạng thái khác trả `409`.
2. Chủ trọ gọi `/confirm` với `confirmedAmount` — lớn hơn 0 và không vượt phần còn phải trả, sai trả `422`; server cộng vào `paidAmount`. Đủ tổng hóa đơn → `DaThanhToan`. Còn thiếu → `QuaHan` nếu đã qua `dueDate`, ngược lại `ThanhToanMotPhan`.
3. Chủ trọ gọi `/reject` → hóa đơn về `QuaHan` nếu đã qua `dueDate`; ngược lại về `ThanhToanMotPhan` nếu `paidAmount` > 0; còn lại về `ChuaThanhToan`.

Chỉ Chủ trọ sở hữu mới xác nhận được thanh toán; mọi vai trò khác trả `403` (BR-06b). Thao tác xác nhận ghi `audit_logs`.

**Quá hạn:** tác vụ định kỳ của hệ thống gắn cờ `QuaHan` cho hóa đơn ở `ChuaThanhToan` hoặc `ThanhToanMotPhan` khi đã sang ngày sau `dueDate` (giờ Việt Nam), và gửi thông báo mức Cao cho cả hai bên. Không có endpoint cho việc này.

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
| `POST` | `/api/v1/contracts/{id}/move-out-notice/withdraw` | Bên đã gửi thông báo | Rút thông báo trả phòng |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice` | Landlord (chủ sở hữu) | Lập hóa đơn thanh lý ở `Nhap` |
| `PUT` | `/api/v1/contracts/{id}/settlement-invoice` | Landlord (chủ sở hữu) | Sửa khi còn ở `Nhap` |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice/send` | Landlord (chủ sở hữu) | Gửi cho người thuê xác nhận |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice/confirm` | Tenant (người đứng tên) | Đồng ý bảng thanh lý |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice/request-changes` | Tenant (người đứng tên) | Chưa đồng ý, bắt buộc có `reason` |
| `POST` | `/api/v1/contracts/{id}/settlement-invoice/finalize` | Landlord (chủ sở hữu) | Tự chốt khi người thuê không phản hồi quá 7 ngày, bắt buộc có `note` |
| `POST` | `/api/v1/contracts/{id}/settlement/complete` | Landlord (chủ sở hữu) | Xác nhận hoàn tất thanh lý |

**`POST /move-out-notice`** — body gồm `expectedMoveOutDate` và `reason`. Chỉ nhận khi hợp đồng ở `DangHieuLuc` đã tới `startDate`, hoặc ở `SapHetHan`; trước `startDate` thì dùng `/cancel`, trạng thái khác trả `409`. `expectedMoveOutDate` không được trước hôm nay, sai trả `422`. Hợp đồng chuyển `DangThanhLy`; server ghi bên gửi vào `move_out_notice_by_user_id` và lý do vào `termination_reason`; bên còn lại nhận thông báo mức Cao. Thông báo gửi trước ít hơn 30 ngày vẫn được chấp nhận (FR-87); response ghi rõ số ngày báo trước để hai bên thấy.

**`POST /move-out-notice/withdraw`** — chỉ bên đã gửi thông báo được gọi, khi hợp đồng ở `DangThanhLy` và chưa có hóa đơn thanh lý; ngược lại trả `409`. Server xóa thông tin thông báo trả phòng, đưa hợp đồng về `SapHetHan` nếu còn 15 ngày hoặc ít hơn tới `endDate`, ngược lại về `DangHieuLuc`; bên còn lại nhận thông báo mức Cao (FR-98).

**`POST /settlement-invoice`** tạo hóa đơn `type = "ThanhLy"` ở `Nhap`, gồm chỉ số điện nước lần cuối và các dòng chi tiết. Chỉ nhận khi hợp đồng ở `DangThanhLy`, chưa có hóa đơn thanh lý, và `moveOutDate` không sau hôm nay — bảng thanh lý lập vào hoặc sau ngày trả phòng thực tế, khi đã chốt số lần cuối; ngược lại trả `409` (FR-54). Kỳ của hóa đơn thanh lý (FR-93):

- `moveOutDate` thuộc tháng ngay sau kỳ định kỳ cuối cùng (hợp đồng chưa có hóa đơn định kỳ thì thuộc tháng của `startDate`): hóa đơn tính từ đầu khoảng đó tới `moveOutDate`.
- `moveOutDate` thuộc chính tháng của kỳ định kỳ cuối cùng — hóa đơn tháng đó đã lập trước thông báo, hoặc người thuê dọn đi sớm hơn dự kiến: hóa đơn thanh lý không có tiền phòng và phí dịch vụ, chỉ tính điện nước từ lần chốt gần nhất.
- Trường hợp khác trả `409` — Chủ trọ lập trước hóa đơn định kỳ của các tháng còn thiếu. Hợp đồng còn hóa đơn định kỳ ở `Nhap` cũng trả `409`. Người thuê ở lại quá tháng của `expectedMoveOutDate` thì tháng đó bị chặn lập hóa đơn định kỳ; bên gửi rút thông báo rồi gửi lại với ngày mới.

```json
{
  "currentElectricityIndex": 1310.0,
  "currentWaterIndex": 89.0,
  "moveOutDate": "2026-12-15",
  "lines": [
    { "category": "BoiThuongHuHong", "description": "Vo kinh cua so", "amount": 300000, "evidencePath": "..." }
  ]
}
```

Mỗi khoản khấu trừ **bắt buộc** là một dòng riêng có `description`; gửi một khoản gộp không mô tả trả `422` (BR-22).

**Dòng do server tự thêm (FR-55, FR-92):**

- `CongNoKyTruoc`: mỗi hóa đơn còn thiếu tiền của hợp đồng một dòng, `amount` = phần còn phải trả, `relatedInvoiceId` trỏ về hóa đơn đó; hóa đơn gốc chuyển `DaChuyenThanhLy`.
- `KhauTruTienCoc`: `amount` = −`depositAmount` của hợp đồng (không thêm khi tiền cọc bằng 0).

Client gửi một trong hai loại dòng này trả `422`. Hợp đồng còn `payment_reports` ở `ChoXacNhan` trả `409` — Chủ trọ xác nhận hoặc từ chối trước.

Dòng `PhiPhat` chỉ được nhận khi người thuê là bên gửi thông báo trả phòng, số ngày báo trước — từ ngày gửi tới `expectedMoveOutDate` — ít hơn 30, và `expectedMoveOutDate` trước `endDate`; ngược lại trả `422`. Tổng các dòng `PhiPhat` không được vượt `depositAmount`, vượt trả `422` (BR-22, FR-87).

`totalAmount` âm nghĩa là Chủ trọ phải hoàn lại phần cọc dư.

**Gửi và xác nhận (FR-57):** `PUT /settlement-invoice` chỉ nhận khi hóa đơn ở `Nhap`, body giống lúc tạo; server tính lại toàn bộ số tiền. `/send` chuyển `Nhap` → `ChoNguoiThueXacNhan`, ghi `sentAt`, người thuê nhận thông báo mức Cao. Người thuê gọi:

- `/request-changes` với `reason` (bắt buộc) → hóa đơn về `Nhap`, Chủ trọ nhận thông báo mức Cao kèm lý do;
- `/confirm` → hóa đơn bị khóa. `totalAmount` > 0 thì sang `ChuaThanhToan` với `issuedAt` = lúc đồng ý, Người thuê thấy mã VietQR theo Mục 9.1 và thanh toán như hóa đơn thường. `totalAmount` < 0 thì sang `ChoHoanCoc`. `totalAmount` = 0 thì sang `DaThanhToan`.

Hai endpoint của người thuê chỉ nhận khi hóa đơn ở `ChoNguoiThueXacNhan`, trạng thái khác trả `409`. Người thuê không thấy hóa đơn thanh lý khi nó còn ở `Nhap`.

**Tự chốt (FR-95):** `/finalize` chỉ nhận khi hóa đơn ở `ChoNguoiThueXacNhan` và đã quá 7 ngày kể từ `sentAt`, ngược lại trả `409`; body có `note` bắt buộc. Kết quả giống `/confirm`. Thao tác ghi `audit_logs` và gửi thông báo mức Cao cho người thuê.

**`POST /settlement/complete`** chỉ thực hiện được khi bảng thanh lý đã khóa — người thuê đồng ý hoặc Chủ trọ tự chốt — và:

- `totalAmount` dương: không cần đã trả đủ. Phần chưa trả giữ nguyên trên hóa đơn thanh lý như một khoản nợ — người thuê vẫn báo thanh toán được, hóa đơn vẫn bị gắn cờ quá hạn (FR-96);
- `totalAmount` âm: hóa đơn ở `ChoHoanCoc`, body có `refundedAt` và `refundMethod`; số tiền hoàn do server đặt bằng −`totalAmount`, lưu vào `deposit_refunded_*` (FR-86), hóa đơn chuyển `DaThanhToan`;
- `totalAmount` bằng 0: hóa đơn đã `DaThanhToan`.

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

**Cách tính:**

- **Doanh thu tháng** của Chủ trọ: tổng `confirmedAmount` của các lượt báo thanh toán được xác nhận trong tháng đó, theo thời điểm xác nhận và giờ Việt Nam. Tiền cọc không phải doanh thu.
- **Hóa đơn chưa thu**: số hóa đơn và tổng phần còn phải trả của các hóa đơn ở `ChuaThanhToan`, `ChoXacNhan`, `ThanhToanMotPhan`, `QuaHan`. Không tính `Nhap` và `DaChuyenThanhLy` — phần nợ của hóa đơn đã chuyển nằm trong hóa đơn thanh lý.
- **Tổng đã thanh toán** của Người thuê: tổng số tiền đã được xác nhận thu trên các hóa đơn của mình, không tính tiền cọc.

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
- Khi nhận `path` trong request nghiệp vụ, server kiểm tra đường dẫn khớp đúng định dạng ở trên — đúng bucket, đúng thư mục của `purpose` tương ứng, nằm trong thư mục của chính người gọi, tên file là 32 ký tự hex và phần mở rộng hợp lệ — rồi kiểm tra file thực sự tồn tại trên Storage; sai thì trả `422`. Không nhận đường dẫn hay URL tùy ý từ client.
- Giới hạn 20 lần tải mỗi giờ cho mỗi tài khoản (Thiết kế An toàn mục 6).
