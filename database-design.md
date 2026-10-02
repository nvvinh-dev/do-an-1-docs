# Thiết kế Cơ sở dữ liệu — Phase 1

Tài liệu chính thức về thiết kế database của hệ thống SmartRent.

**Phạm vi:** Phase 1 theo phân kỳ trong [README](../README.md) — BP-01 (đăng ký, xác thực, duyệt Chủ trọ), BP-02 (khu trọ và phòng trọ), BP-03 (đăng/ẩn tin), BP-04 (tìm kiếm bằng bộ lọc), BP-06 (yêu cầu thuê, đặt cọc, hợp đồng), BP-07 (hóa đơn và thanh toán), BP-10 (chấm dứt hợp đồng và tất toán cọc), thông báo trong ứng dụng, nhật ký hệ thống và dashboard cơ bản.

Các bảng phục vụ Phase 2 và Phase 3 (sự cố, ở ghép, khiếu nại, lịch xem phòng, gia hạn) **không** nằm trong tài liệu này.

---

## 1. Quy ước chung

| Hạng mục | Quy ước |
|---|---|
| **Khóa chính** | `id`, kiểu `bigint`, tự tăng |
| **Khóa ngoại** | `<tên_bảng_số_ít>_id`, ví dụ `room_id`, `contract_id` |
| **Đặt tên** | `snake_case` cho cả tên bảng và tên cột; tên bảng ở dạng số nhiều |
| **Trạng thái** | Lưu dưới dạng chuỗi text đúng tên trạng thái, ví dụ `DangThue`, `ChuaThanhToan`. Mỗi cột trạng thái có ràng buộc `CHECK` giới hạn trong tập giá trị hợp lệ |
| **Tiền** | `numeric(14,2)`. Số tiền do server tính được làm tròn đến đồng (`MidpointRounding.AwayFromZero`) |
| **Chỉ số điện nước** | `numeric(12,2)` |
| **Thời điểm** | `timestamptz` |
| **Ngày** | `date` khi chỉ cần ngày (ngày bắt đầu hợp đồng, ngày trả phòng) |
| **File** | Các cột `*_url` lưu đường dẫn nội bộ `path` do `POST /files` trả về (dạng `bucket/purpose/...`), không lưu URL. URL để xem được sinh ra lúc đọc — URL có chữ ký, có hạn với file riêng tư |

**Về cột thời gian:** tài liệu này chỉ đưa vào các cột thời gian **có ý nghĩa nghiệp vụ** (`submitted_at`, `issued_at`, `deposit_received_at`...). Không thêm `created_at` / `updated_at` / `created_by` / `deleted_at` một cách mặc định cho mọi bảng. Nhu cầu truy vết ai thao tác lúc nào được đáp ứng bằng bảng `audit_logs` theo BR-23.

**Về xóa dữ liệu:** không có cột `is_deleted`. Theo BR-09 và QR-02, phòng và khu trọ đã phát sinh hợp đồng hoặc hóa đơn được chuyển sang trạng thái `LuuTru`; dữ liệu tài chính không bao giờ bị xóa.

---

## 2. Tổng quan quan hệ

```
users ──┬──< landlord_applications
        │
        ├──< properties ──< rooms ──┬──< room_service_fees
        │        │                  ├──< room_images
        │        └──< property_images├──< room_amenities >── amenities
        │                            │
        │                            ├──< rental_requests
        │                            └──< contracts ──┬──< contract_service_fees
        │                                             ├──< contract_occupants
        │                                             └──< invoices ──┬──< invoice_lines
        │                                                             └──< payment_reports
        ├──< notifications
        └──< audit_logs
```

---

## 3. Danh tính và phân quyền

### 3.1 `users`

Do ASP.NET Identity quản lý, đặt lại tên bảng và cột theo `snake_case`, khóa chính kiểu `bigint`.

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `email` | text | NOT NULL, UNIQUE | Dùng để đăng nhập |
| `password_hash` | text | NOT NULL | Do Identity quản lý |
| `full_name` | text | NOT NULL | |
| `phone_number` | text | | |
| `phone_number_confirmed` | boolean | NOT NULL | `true` khi Admin đã xác minh số điện thoại lúc duyệt hồ sơ Chủ trọ (BR-01); trở về `false` khi người dùng đổi số điện thoại |
| `is_locked` | boolean | NOT NULL | Admin khóa tài khoản |
| `lock_reason` | text | | Bắt buộc khi `is_locked` = true |
| `registered_at` | timestamptz | NOT NULL | |
| `bank_bin` | text | | Mã BIN 6 chữ số của ngân hàng theo chuẩn VietQR — tài khoản nhận tiền của Chủ trọ (BR-26) |
| `bank_account_number` | text | | Số tài khoản nhận tiền |
| `bank_account_name` | text | | Tên chủ tài khoản |

**Tài khoản nhận tiền (BR-26):** ba cột `bank_*` chỉ có ý nghĩa với người có vai trò `Landlord`, và **cùng có giá trị hoặc cùng NULL** — ràng buộc `CHECK` ở mức database. Mỗi Chủ trọ có tối đa một tài khoản nhận tiền, dùng chung cho mọi khu trọ. Việc khai báo không bắt buộc; các cột NULL thì hệ thống không hiển thị mã VietQR. Mỗi lần khai báo hoặc sửa đều ghi `audit_logs` với giá trị cũ và mới (BR-23), để tra được tài khoản nhận tiền tại một thời điểm bất kỳ khi có tranh chấp.

Các cột kỹ thuật khác của Identity (`security_stamp`, `concurrency_stamp`, `access_failed_count`...) giữ nguyên mặc định, không liệt kê ở đây.

### 3.2 `roles` và `user_roles`

Do ASP.NET Identity quản lý. Ba vai trò: `Admin`, `Landlord`, `Tenant`.

Mỗi tài khoản có đúng một vai trò. Người dùng mới đăng ký mặc định nhận vai trò `Tenant` (BP-01). Vai trò `Landlord` chỉ được cấp sau khi Admin duyệt hồ sơ, và thay cho vai trò `Tenant` (BR-01).

### 3.3 `landlord_applications`

Hồ sơ đăng ký làm Chủ trọ.

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `user_id` | bigint | FK → `users.id`, NOT NULL | Người nộp hồ sơ |
| `id_card_number` | text | NOT NULL | Số CCCD |
| `id_card_front_url` | text | NOT NULL | Ảnh mặt trước |
| `id_card_back_url` | text | NOT NULL | Ảnh mặt sau |
| `ownership_document_url` | text | NOT NULL | Giấy tờ chứng minh quyền sở hữu/quản lý |
| `status` | text | NOT NULL, CHECK | `ChoDuyet` / `DaDuyet` / `TuChoi` / `DaThuHoi` |
| `submitted_at` | timestamptz | NOT NULL | |
| `reviewed_by_user_id` | bigint | FK → `users.id` | Admin đã xử lý |
| `reviewed_at` | timestamptz | | |
| `reject_reason` | text | | Bắt buộc khi `status` = `TuChoi` |

**Ràng buộc nghiệp vụ:** mỗi `user_id` chỉ có tối đa một hồ sơ ở trạng thái `ChoDuyet` tại một thời điểm. Hồ sơ bị từ chối được bổ sung và nộp lại thành bản ghi mới.

**Ảnh giấy tờ (QR-04):** `id_card_front_url`, `id_card_back_url`, `ownership_document_url` chỉ được trả về cho Admin trong quá trình duyệt. Không endpoint nào khác được expose các cột này.

---

## 4. Khu trọ và phòng trọ

### 4.1 `properties` — Khu trọ

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `landlord_user_id` | bigint | FK → `users.id`, NOT NULL | BR-02: một khu trọ thuộc một Chủ trọ |
| `name` | text | NOT NULL | |
| `address` | text | NOT NULL | Địa chỉ đầy đủ |
| `ward` | text | | Phường/xã — dùng cho bộ lọc khu vực |
| `district` | text | | Quận/huyện — dùng cho bộ lọc khu vực |
| `city` | text | NOT NULL | Tỉnh/thành — dùng cho bộ lọc khu vực |
| `description` | text | | |
| `status` | text | NOT NULL, CHECK | `DangKhaiThac` / `LuuTru` |

**BR-10:** không được chuyển `status` sang `LuuTru` khi còn phòng có `occupancy_status` là `DangGiuCho` hoặc `DangThue`.

### 4.2 `rooms` — Phòng trọ

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `property_id` | bigint | FK → `properties.id`, NOT NULL | BR-02 |
| `code` | text | NOT NULL | Mã/tên phòng, duy nhất trong một khu trọ |
| `area` | numeric(8,2) | NOT NULL | Diện tích m² |
| `max_occupants` | int | NOT NULL | Số người tối đa (BR-11) |
| `rent_price` | numeric(14,2) | NOT NULL | Giá thuê/tháng hiện tại |
| `electricity_unit_price` | numeric(14,2) | NOT NULL | Đơn giá điện hiện tại |
| `water_unit_price` | numeric(14,2) | NOT NULL | Đơn giá nước hiện tại |
| `description` | text | | |
| `occupancy_status` | text | NOT NULL, CHECK | `Trong` / `DangGiuCho` / `DangThue` / `BaoTri` / `LuuTru` |
| `visibility_status` | text | NOT NULL, CHECK | `DangHienThi` / `DaAnBoiChuTro` / `DaAnBoiAdmin` |

**Hai chiều trạng thái là độc lập.** `occupancy_status` mô tả tình trạng khai thác thực tế; `visibility_status` mô tả việc phòng có được quảng bá hay không. Phòng mới tạo có `visibility_status = 'DaAnBoiChuTro'` cho tới khi Chủ trọ bật hiển thị (BP-03). Giá trị `DaAnBoiAdmin` giữ sẵn trong ràng buộc nhưng chỉ phát sinh từ Phase 2, khi Admin ẩn tin vi phạm cùng BP-13.

**Chuyển `occupancy_status` thủ công:** Chủ trọ chỉ chuyển được giữa `Trong` và `BaoTri`, và sang `LuuTru` khi lưu trữ. `DangGiuCho` và `DangThue` do hệ thống đặt theo yêu cầu thuê, hợp đồng và thanh lý.

**BR-05 — điều kiện xuất hiện trong kết quả tìm kiếm.** Phòng chỉ hiển thị khi thỏa mãn đồng thời: `occupancy_status = 'Trong'`, `visibility_status = 'DangHienThi'`, khu trọ có `status = 'DangKhaiThac'`, và Chủ trọ sở hữu có `is_locked = false`.

**BR-08:** chỉ được chuyển `occupancy_status` về `Trong` khi hợp đồng hiện tại của phòng đã ở `DaThanhLy` hoặc `DaHuy`.

**BR-09:** phòng đã từng phát sinh hợp đồng hoặc hóa đơn không được xóa, chỉ chuyển `occupancy_status` sang `LuuTru`.

**BR-12:** ba cột giá ở bảng này là giá **hiện hành của phòng**, dùng khi lập hợp đồng mới. Sửa các cột này không ảnh hưởng tới hợp đồng đang hiệu lực và hóa đơn đã phát hành.

### 4.3 `room_service_fees` — Phí dịch vụ cố định của phòng

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `room_id` | bigint | FK → `rooms.id`, NOT NULL | |
| `name` | text | NOT NULL | Rác, internet, giữ xe, phí quản lý... |
| `amount` | numeric(14,2) | NOT NULL | Số tiền mỗi kỳ |

### 4.4 `amenities`, `property_amenities`, `room_amenities` — Tiện ích

Tiện ích được chuẩn hóa thành danh mục vì bộ lọc tìm kiếm ở BP-04 cho phép lọc theo tiện ích.

`amenities`: `id` (PK), `name` (NOT NULL, UNIQUE), `scope` (`KhuTro` / `Phong`).

`property_amenities`: `property_id` + `amenity_id`, khóa chính tổ hợp.

`room_amenities`: `room_id` + `amenity_id`, khóa chính tổ hợp.

### 4.5 `property_images`, `room_images`

Cùng cấu trúc: `id` (PK), khóa ngoại tới khu trọ hoặc phòng, `url` (NOT NULL), `display_order` (int).

---

## 5. Yêu cầu thuê và hợp đồng

### 5.1 `rental_requests` — Yêu cầu thuê

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `room_id` | bigint | FK → `rooms.id`, NOT NULL | |
| `tenant_user_id` | bigint | FK → `users.id`, NOT NULL | |
| `expected_move_in_date` | date | NOT NULL | |
| `expected_occupants` | int | NOT NULL | |
| `note` | text | | |
| `status` | text | NOT NULL, CHECK | `ChoDuyet` / `DaDuyet` / `DaLapHopDong` / `TuChoi` / `DaHuy` / `HetHan` |
| `submitted_at` | timestamptz | NOT NULL | |
| `processed_at` | timestamptz | | Thời điểm Chủ trọ duyệt hoặc từ chối |
| `reject_reason` | text | | Bắt buộc khi `status` = `TuChoi` |

**BR-06:** khi một yêu cầu của phòng được duyệt, toàn bộ yêu cầu khác của cùng phòng đang ở `ChoDuyet` phải tự động chuyển sang `TuChoi` với `reject_reason` do hệ thống sinh.

**BR-27:** mỗi cặp (`room_id`, `tenant_user_id`) chỉ có tối đa một yêu cầu ở `ChoDuyet` — unique index có điều kiện.

**Hết hạn:** yêu cầu ở `ChoDuyet` quá 7 ngày kể từ `submitted_at` chuyển sang `HetHan`. Yêu cầu ở `DaDuyet` quá 3 ngày kể từ `processed_at` mà chưa được lập hợp đồng cũng chuyển sang `HetHan` (BP-06 A3).

**Đã lập hợp đồng:** yêu cầu chuyển sang `DaLapHopDong` ngay khi hợp đồng được tạo từ nó. Hợp đồng bị hủy sau đó không làm đổi trạng thái này.

### 5.2 `contracts` — Hợp đồng

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `room_id` | bigint | FK → `rooms.id`, NOT NULL | |
| `tenant_user_id` | bigint | FK → `users.id`, NOT NULL | BR-11: một người đứng tên |
| `rental_request_id` | bigint | FK → `rental_requests.id` | Nguồn gốc hợp đồng |
| `rent_price` | numeric(14,2) | NOT NULL | Giá đã chốt (BR-12) |
| `electricity_unit_price` | numeric(14,2) | NOT NULL | Đơn giá điện đã chốt |
| `water_unit_price` | numeric(14,2) | NOT NULL | Đơn giá nước đã chốt |
| `deposit_amount` | numeric(14,2) | NOT NULL | BR-21: bắt buộc ghi nhận, có thể bằng 0 |
| `initial_electricity_index` | numeric(12,2) | NOT NULL | Chỉ số điện lúc bàn giao phòng — chỉ số cũ của hóa đơn đầu tiên (BR-14) |
| `initial_water_index` | numeric(12,2) | NOT NULL | Chỉ số nước lúc bàn giao phòng |
| `start_date` | date | NOT NULL | |
| `end_date` | date | NOT NULL | |
| `payment_due_days` | int | NOT NULL | Số ngày được phép thanh toán kể từ khi phát hành |
| `status` | text | NOT NULL, CHECK | `Nhap` / `ChoNguoiThueXacNhan` / `ChoNhanCoc` / `DangHieuLuc` / `SapHetHan` / `DangThanhLy` / `DaThanhLy` / `DaHuy` |
| `tenant_confirmed_at` | timestamptz | | Người thuê đồng ý điều khoản |
| `deposit_received_at` | timestamptz | | Thời điểm nhận cọc thực tế, do Chủ trọ nhập khi xác nhận đã nhận cọc; không sau thời điểm xác nhận |
| `deposit_received_method` | text | | Hình thức nhận cọc |
| `deposit_refunded_amount` | numeric(14,2) | | Số tiền cọc đã hoàn. Server tính khi Chủ trọ hủy (toàn bộ cọc) và khi hóa đơn thanh lý có số dư âm (phần cọc dư); Chủ trọ nhập, trong khoảng 0 tới `deposit_amount`, khi Người thuê hủy (BR-22) |
| `deposit_refunded_at` | timestamptz | | Thời điểm Chủ trọ hoàn cọc |
| `deposit_refund_method` | text | | Hình thức hoàn cọc |
| `deposit_refund_note` | text | | Lý do giữ lại cọc — bắt buộc khi Người thuê hủy và số hoàn nhỏ hơn tiền cọc |
| `activated_at` | timestamptz | | Thời điểm chuyển sang `DangHieuLuc` |
| `move_out_notice_at` | timestamptz | | Thời điểm gửi thông báo trả phòng |
| `expected_move_out_date` | date | | Ngày trả phòng dự kiến |
| `terminated_at` | timestamptz | | Thời điểm hoàn tất thanh lý |
| `termination_reason` | text | | |
| `cancel_reason` | text | | Bắt buộc khi `status` = `DaHuy` |
| `cancelled_by_user_id` | bigint | FK → `users.id` | Bên hủy hợp đồng; NULL khi hệ thống tự hủy do hết hạn giữ chỗ |

Trạng thái `DaKetThucGiaHan` thuộc BP-09 (Phase 3), chưa đưa vào tập giá trị hợp lệ của Phase 1.

**BR-07:** tại một thời điểm, một `room_id` chỉ có tối đa một hợp đồng ở `DangHieuLuc`, `SapHetHan` hoặc `DangThanhLy`. Ràng buộc này được bảo đảm bằng unique index có điều kiện trên `room_id`.

**BR-21:** chỉ chuyển sang `DangHieuLuc` khi `tenant_confirmed_at` đã có giá trị và, nếu `deposit_amount` > 0, `deposit_received_at` cũng đã có giá trị. Xác nhận cọc chỉ diễn ra ở `ChoNhanCoc`; `deposit_amount` = 0 thì hợp đồng đi thẳng từ `ChoNguoiThueXacNhan` sang `DangHieuLuc`.

**Chỉ số đầu (BR-14):** `initial_electricity_index` và `initial_water_index` được nhập khi lập hợp đồng và là chỉ số cũ của hóa đơn đầu tiên. Khi hợp đồng còn ở `Nhap`, hai cột này sửa cùng các điều khoản khác. Khi hợp đồng đã ở `DangHieuLuc` mà chưa có hóa đơn nào khác `DaHuy`, Chủ trọ vẫn sửa được — dùng cho trường hợp số thực tế lúc bàn giao khác số đã ghi; mỗi lần sửa ghi `audit_logs` (BR-23) và thông báo cho người thuê.

**Hạn giữ chỗ (BP-06 A3):** quá 3 ngày kể từ `rental_requests.processed_at` của yêu cầu gốc mà hợp đồng chưa ở `DangHieuLuc` thì hợp đồng chuyển sang `DaHuy`, phòng trở lại `Trong`.

**Hủy trước ngày bắt đầu (BR-22):** hợp đồng ở `DangHieuLuc` nhưng chưa tới `start_date` được một trong hai bên chuyển sang `DaHuy`, ghi `cancelled_by_user_id`, phòng trở lại `Trong` — không tạo bản ghi `invoices` nào cho trường hợp này. Nếu đã có `deposit_received_at`: Chủ trọ hủy thì số hoàn bằng toàn bộ cọc; Người thuê hủy thì Chủ trọ nhập số hoàn từ 0 tới `deposit_amount`, kèm `deposit_refund_note` khi số hoàn nhỏ hơn. Việc hoàn cọc được Chủ trọ ghi nhận ở một bước riêng sau khi hủy, vào các cột `deposit_refund*`. Đây là transition `DangHieuLuc` → `DaHuy` duy nhất được phép; trong Phase 1, hợp đồng đã qua `start_date` chỉ kết thúc qua `DangThanhLy` → `DaThanhLy`.

**Quá `end_date`:** hợp đồng chưa có thông báo trả phòng vẫn giữ trạng thái hiện tại và tiếp tục hiệu lực theo điều khoản đã chốt; hóa đơn định kỳ vẫn được lập cho tới khi một bên gửi thông báo trả phòng.

### 5.3 `contract_service_fees` — Phí dịch vụ đã chốt trong hợp đồng

`id` (PK), `contract_id` (FK, NOT NULL), `name` (NOT NULL), `amount` (numeric(14,2), NOT NULL).

Đây là **bản sao tại thời điểm lập hợp đồng** của `room_service_fees`, không phải tham chiếu. Sửa phí ở mức phòng sau đó không ảnh hưởng hợp đồng đang hiệu lực (BR-12).

### 5.4 `contract_occupants` — Người ở cùng

`id` (PK), `contract_id` (FK, NOT NULL), `full_name` (NOT NULL), `phone_number`.

Theo BR-11, người ở cùng chỉ được ghi nhận thông tin, không có tài khoản và không có nghĩa vụ tài chính. Tổng số người ở (người đứng tên + người ở cùng) không vượt `rooms.max_occupants`.

---

## 6. Hóa đơn và thanh toán

### 6.1 `invoices` — Hóa đơn

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `contract_id` | bigint | FK → `contracts.id`, NOT NULL | |
| `type` | text | NOT NULL, CHECK | `DinhKy` / `ThanhLy` |
| `period_start` | date | NOT NULL | Ngày đầu kỳ — do server xác định |
| `period_end` | date | NOT NULL | Ngày cuối kỳ — do server xác định |
| `previous_electricity_index` | numeric(12,2) | NOT NULL | |
| `current_electricity_index` | numeric(12,2) | NOT NULL | |
| `electricity_unit_price` | numeric(14,2) | NOT NULL | Bản sao đơn giá đã áp dụng (BR-13) |
| `electricity_amount` | numeric(14,2) | NOT NULL | |
| `previous_water_index` | numeric(12,2) | NOT NULL | |
| `current_water_index` | numeric(12,2) | NOT NULL | |
| `water_unit_price` | numeric(14,2) | NOT NULL | Bản sao đơn giá đã áp dụng (BR-13) |
| `water_amount` | numeric(14,2) | NOT NULL | |
| `rent_amount` | numeric(14,2) | NOT NULL | Tính theo tỷ lệ ngày ở với kỳ đầu/kỳ cuối (BR-15) |
| `service_fee_amount` | numeric(14,2) | NOT NULL | Tổng phí dịch vụ của kỳ, cũng tính theo tỷ lệ ngày ở với kỳ không trọn tháng (BR-15) |
| `total_amount` | numeric(14,2) | NOT NULL | Tổng cộng, bao gồm các dòng ở `invoice_lines` |
| `paid_amount` | numeric(14,2) | NOT NULL | Số tiền đã thu được xác nhận |
| `status` | text | NOT NULL, CHECK | `Nhap` / `ChuaThanhToan` / `ChoXacNhan` / `ThanhToanMotPhan` / `QuaHan` / `DaThanhToan` / `DaHuy` / `DaChuyenThanhLy` / `ChoNguoiThueXacNhan` / `ChoHoanCoc` |
| `electricity_meter_photo_url` | text | | Ảnh chụp đồng hồ điện |
| `water_meter_photo_url` | text | | Ảnh chụp đồng hồ nước |
| `issued_at` | timestamptz | | Thời điểm phát hành |
| `due_date` | date | | Hạn thanh toán, tính từ `issued_at` và `payment_due_days` |
| `settled_at` | timestamptz | | Thời điểm chuyển sang `DaThanhToan` |
| `tenant_confirmed_at` | timestamptz | | Chỉ với `ThanhLy`: thời điểm người thuê đồng ý bảng thanh lý |
| `change_request_reason` | text | | Chỉ với `ThanhLy`: lý do người thuê chưa đồng ý ở lần gần nhất |
| `cancel_reason` | text | | Bắt buộc khi `status` = `DaHuy` |

**BR-14:** `current_electricity_index` ≥ `previous_electricity_index` và `current_water_index` ≥ `previous_water_index` — ràng buộc `CHECK` ở mức database. `previous_*` của một kỳ bắt buộc bằng `current_*` của kỳ liền trước của cùng hợp đồng; kỳ đầu tiên lấy `contracts.initial_*_index`. "Kỳ liền trước" bỏ qua các hóa đơn `DaHuy`.

**Kỳ hóa đơn (BR-15, BR-17):** mỗi kỳ là một tháng dương lịch, do server xác định — client không gửi `period_start`, `period_end`. Kỳ đầu tiên chạy từ `contracts.start_date` tới cuối tháng đó; mỗi kỳ sau là tháng liền sau kỳ chưa hủy gần nhất. Tháng có ngày trả phòng không có hóa đơn định kỳ mà thuộc hóa đơn thanh lý.

**BR-17:** mỗi `contract_id` chỉ có một hóa đơn `type = 'DinhKy'` chưa hủy cho mỗi cặp (`period_start`, `period_end`) — unique index có điều kiện `type = 'DinhKy' AND status <> 'DaHuy'`, để hóa đơn đã hủy không chặn việc lập lại kỳ đó.

**BR-16:** hóa đơn ở `DaThanhToan` không được sửa. Chỉ hóa đơn định kỳ mới nhất chưa hủy của hợp đồng mới được sửa hoặc hủy — sửa một hóa đơn cũ hơn sẽ làm gãy chuỗi chỉ số của BR-14. Sai sót ở hóa đơn không còn sửa được điều chỉnh bằng một dòng `DieuChinhKhac` có `related_invoice_id` trỏ về hóa đơn gốc, đặt ở hóa đơn kỳ kế tiếp hoặc hóa đơn thanh lý.

**Hóa đơn thanh lý (BP-10):** `type = 'ThanhLy'`, các khoản cộng thêm và khoản trừ tiền cọc nằm ở `invoice_lines`. `total_amount` có thể âm — khi đó Chủ trọ phải hoàn lại phần cọc dư.

**Vòng đời hóa đơn thanh lý (FR-57):** `Nhap` → `ChoNguoiThueXacNhan` khi Chủ trọ gửi; người thuê chưa đồng ý thì về `Nhap`, ghi `change_request_reason`. Người thuê đồng ý thì ghi `tenant_confirmed_at`, hóa đơn bị khóa và:

- `total_amount` > 0 → `ChuaThanhToan`, `issued_at` = lúc đồng ý, rồi đi luồng thanh toán như hóa đơn định kỳ;
- `total_amount` < 0 → `ChoHoanCoc`, sang `DaThanhToan` khi Chủ trọ ghi nhận hoàn cọc lúc hoàn tất thanh lý;
- `total_amount` = 0 → `DaThanhToan`.

Với hóa đơn thanh lý, `DaThanhToan` nghĩa là đã tất toán xong.

**Kết chuyển công nợ (FR-92):** dòng `CongNoKyTruoc` và `KhauTruTienCoc` do server sinh khi lập hóa đơn thanh lý. Mỗi hóa đơn còn nợ của hợp đồng (`ChuaThanhToan`, `ThanhToanMotPhan`, `QuaHan`) thành một dòng `CongNoKyTruoc` bằng `total_amount − paid_amount`, `related_invoice_id` trỏ về nó, và hóa đơn đó chuyển sang `DaChuyenThanhLy` trong cùng transaction.

### 6.2 `invoice_lines` — Dòng chi tiết của hóa đơn

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `invoice_id` | bigint | FK → `invoices.id`, NOT NULL | |
| `category` | text | NOT NULL, CHECK | `CongNoKyTruoc` / `BoiThuongHuHong` / `PhiPhat` / `KhauTruTienCoc` / `DieuChinhKhac` |
| `description` | text | NOT NULL | Lý do của khoản mục — bắt buộc theo BR-22 |
| `amount` | numeric(14,2) | NOT NULL | Dương là khoản phải thu, âm là khoản trừ |
| `evidence_url` | text | | Ảnh minh chứng hư hỏng |
| `related_invoice_id` | bigint | FK → `invoices.id` | Hóa đơn gốc, thuộc cùng hợp đồng: bắt buộc với `CongNoKyTruoc`; với `DieuChinhKhac` khi dòng đó điều chỉnh sai sót của một hóa đơn trước (BR-16) |

**BR-22:** mọi khoản khấu trừ tiền cọc phải là một dòng riêng có `description`. Không cho phép gộp thành một khoản không giải thích.

### 6.3 `payment_reports` — Người thuê báo đã thanh toán

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `invoice_id` | bigint | FK → `invoices.id`, NOT NULL | |
| `reported_amount` | numeric(14,2) | NOT NULL | Số tiền người thuê khai đã trả |
| `proof_image_url` | text | NOT NULL | Minh chứng — bắt buộc theo BR-06b |
| `reported_at` | timestamptz | NOT NULL | |
| `status` | text | NOT NULL, CHECK | `ChoXacNhan` / `DaXacNhan` / `TuChoiXacNhan` |
| `confirmed_by_user_id` | bigint | FK → `users.id` | Chỉ Chủ trọ (BR-06b) |
| `confirmed_at` | timestamptz | | |
| `confirmed_amount` | numeric(14,2) | | Số tiền Chủ trọ xác nhận thực thu |
| `reject_reason` | text | | Bắt buộc khi `status` = `TuChoiXacNhan` |

Một hóa đơn có thể có nhiều lượt báo thanh toán (bị từ chối rồi báo lại, hoặc trả làm nhiều lần).

**Mã VietQR không có bảng riêng.** Mã được dựng mỗi lần xem từ `users.bank_*` của Chủ trọ cùng số tiền còn phải trả của hóa đơn hoặc tiền cọc của hợp đồng; không lưu lại mã đã hiển thị hay kết quả chuyển khoản (BR-26).

---

## 7. Thông báo và nhật ký

### 7.1 `notifications`

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `recipient_user_id` | bigint | FK → `users.id`, NOT NULL | |
| `event_type` | text | NOT NULL | Mã sự kiện theo danh mục thông báo |
| `title` | text | NOT NULL | |
| `content` | text | NOT NULL | |
| `related_entity_type` | text | | `Contract`, `Invoice`, `RentalRequest`, `LandlordApplication`, `AppUser` |
| `related_entity_id` | bigint | | |
| `is_read` | boolean | NOT NULL | |
| `created_at` | timestamptz | NOT NULL | Thời điểm phát sinh sự kiện |
| `read_at` | timestamptz | | |

Phase 1 chỉ gửi thông báo trong ứng dụng, và chỉ cho các sự kiện ở mức **Cao** theo danh mục sự kiện thông báo.

### 7.2 `audit_logs` — Nhật ký hệ thống

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | bigint | PK | |
| `actor_user_id` | bigint | FK → `users.id`, NOT NULL | Người thực hiện |
| `action` | text | NOT NULL | Loại thao tác |
| `entity_type` | text | NOT NULL | |
| `entity_id` | bigint | NOT NULL | |
| `old_value` | jsonb | | Giá trị trước |
| `new_value` | jsonb | | Giá trị sau |
| `occurred_at` | timestamptz | NOT NULL | |

**BR-23 — các thao tác bắt buộc ghi nhật ký:** thay đổi giá thuê hoặc đơn giá điện nước; tạo, sửa, hủy hóa đơn; nhập hoặc sửa chỉ số điện nước; xác nhận thanh toán; xác nhận nhận cọc và hoàn cọc; khai báo hoặc sửa tài khoản ngân hàng nhận tiền của Chủ trọ; duyệt, từ chối, thu hồi vai trò Chủ trọ; khóa và mở khóa tài khoản; ẩn tin đăng.

**QR-03 — bảng này chỉ được INSERT.** Không có endpoint nào cho phép UPDATE hoặc DELETE, kể cả với vai trò Admin. Ở mức database, trigger `trg_audit_logs_chi_them` chặn mọi lệnh `UPDATE`, `DELETE` và `TRUNCATE` trên bảng này bằng một lỗi — kể cả khi lệnh đến từ code của ứng dụng. Trigger được tạo trong migration nên áp dụng cho mọi database chạy migration.

---

## 8. Chỉ mục

| Bảng | Chỉ mục | Mục đích |
|---|---|---|
| `rooms` | `(occupancy_status, visibility_status)` | Lọc điều kiện hiển thị BR-05 |
| `rooms` | `(property_id)` | Liệt kê phòng theo khu trọ |
| `rooms` | `(rent_price)`, `(area)`, `(max_occupants)` | Bộ lọc tìm kiếm BP-04 |
| `properties` | `(city, district, ward)` | Bộ lọc khu vực |
| `properties` | `(landlord_user_id)` | Phân quyền theo sở hữu BR-04 |
| `contracts` | `(room_id)` unique có điều kiện với trạng thái đang chiếm dụng | BR-07 |
| `contracts` | `(tenant_user_id)` | Người thuê xem hợp đồng của mình |
| `invoices` | `(contract_id, period_start, period_end)` unique khi `type = 'DinhKy'` và `status <> 'DaHuy'` | BR-17 |
| `invoices` | `(status, due_date)` | Quét hóa đơn quá hạn |
| `rental_requests` | `(room_id, status)` | BR-06 |
| `rental_requests` | `(room_id, tenant_user_id)` unique khi `status = 'ChoDuyet'` | BR-27 |
| `notifications` | `(recipient_user_id, is_read)` | Đếm thông báo chưa đọc |
| `audit_logs` | `(entity_type, entity_id)` | Tra cứu khi xử lý khiếu nại |

---

## 9. Những gì chưa thuộc Phase 1

Các bảng sau phục vụ Phase 2 và Phase 3, chưa thiết kế trong tài liệu này: yêu cầu sửa chữa (BP-08), hồ sơ ở ghép và điểm phù hợp (BP-11), khiếu nại (BP-13), lịch xem phòng (BP-05), đề nghị gia hạn (BP-09).
