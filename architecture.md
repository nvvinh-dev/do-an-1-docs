# Kiến trúc phần mềm — Phase 1

Tài liệu chính thức về kiến trúc của hệ thống SmartRent.

**Mục đích:** để hai thành viên đặt code cùng loại vào cùng một chỗ. Khi không chắc một đoạn logic thuộc về tầng nào, tra tài liệu này trước khi viết.

---

## 1. Tổng quan

Hệ thống gồm hai phần chạy độc lập:

```
[ Frontend — React + TypeScript ]
            │  HTTPS, JSON, JWT
            ▼
[ Backend — ASP.NET Core Web API ]
            │  Entity Framework Core
            ▼
[ PostgreSQL — Supabase ]
```

**Frontend không bao giờ kết nối trực tiếp tới database.** Mọi truy cập dữ liệu đi qua Backend API. Đây là ràng buộc bắt buộc, không phải khuyến nghị — nó là điều kiện để mọi kiểm soát an toàn ở backend có tác dụng.

---

## 2. Phân tầng backend

Backend chia thành ba project, đúng theo cấu trúc thư mục trong [README](../README.md):

| Project | Chứa gì | Không chứa gì |
|---|---|---|
| **SmartRent.Api** | Controller, **service điều phối nghiệp vụ**, cấu hình ứng dụng, xác thực JWT, kiểm tra phân quyền, validation dữ liệu đầu vào, rate limiting, ánh xạ giữa dữ liệu vào/ra và entity | Công thức tính tiền, điều kiện chuyển trạng thái — những thứ này thuộc Domain |
| **SmartRent.Domain** | Entity, enum trạng thái, quy tắc nghiệp vụ, điều kiện chuyển trạng thái, công thức tính tiền | Tham chiếu tới EF Core, tới ASP.NET Core, tới bất kỳ tầng nào khác |
| **SmartRent.Infrastructure** | `DbContext`, cấu hình ánh xạ entity sang bảng, migration, truy vấn dữ liệu, gửi thông báo, ghi nhật ký, **gọi dịch vụ ngoài**: Supabase Storage và SMTP | Quy tắc nghiệp vụ |

### 2.1 Quy tắc phụ thuộc

```
Api  ──►  Infrastructure  ──►  Domain
 └──────────────────────────────►┘
```

`Domain` **không phụ thuộc vào tầng nào khác** — mọi thứ hướng vào nó. Cụ thể: không được thêm package EF Core hay ASP.NET Core vào `SmartRent.Domain`. Nếu một đoạn code trong Domain cần tới `DbContext`, đó là dấu hiệu đoạn code đó đặt sai tầng.

**Ngoại lệ duy nhất — tài khoản và vai trò.** `AppUser` và `AppRole` nằm ở `SmartRent.Infrastructure/Identity/` chứ không ở `Domain`, vì chúng kế thừa `IdentityUser<long>` và `IdentityRole<long>` của ASP.NET Identity. Đưa chúng vào `Domain` sẽ kéo theo phụ thuộc vào ASP.NET Core và phá vỡ quy tắc trên. Đây là hai lớp duy nhất được đặt như vậy; mọi entity nghiệp vụ khác đều ở `Domain`.

### 2.2 Service điều phối nghiệp vụ

Các luồng nghiệp vụ của hệ thống gồm nhiều bước phải chạy trong một transaction — duyệt yêu cầu thuê vừa đổi trạng thái yêu cầu, vừa giữ chỗ phòng, vừa tự từ chối các yêu cầu còn lại, vừa gửi thông báo, vừa ghi nhật ký. Phần điều phối đó đặt trong **service**, không đặt trong controller.

Service nằm ở `SmartRent.Api/Services/`, **chia theo vùng nghiệp vụ chứ không theo thực thể**:

| Service | Phụ trách |
|---|---|
| `AuthService` | Đăng ký, đăng nhập, đổi và đặt lại mật khẩu, thông tin cá nhân |
| `TokenService` | Sinh access token khi đăng nhập |
| `LoginAttemptLimiter` | Đếm lần đăng nhập sai theo email + IP, chặn khi vượt ngưỡng |
| `UserAdminService` | Khóa, mở khóa tài khoản |
| `LandlordApplicationService` | Nộp, duyệt, từ chối hồ sơ Chủ trọ |
| `LandlordBankAccountService` | Khai báo, sửa tài khoản ngân hàng nhận tiền của Chủ trọ |
| `PropertyService` | Khu trọ, phòng, trạng thái khai thác và hiển thị |
| `RentalRequestService` | Gửi, duyệt, từ chối, hết hạn yêu cầu thuê |
| `ContractService` | Lập hợp đồng, xác nhận điều khoản, xác nhận cọc, kích hoạt, hủy và ghi nhận hoàn cọc khi hủy |
| `InvoiceService` | Chốt chỉ số, tạo và phát hành hóa đơn, ghi nhận thanh toán |
| `SettlementService` | Thông báo trả phòng, hóa đơn thanh lý, tất toán cọc |

**Phân vai rõ ràng:**

- **Controller** chỉ lo HTTP: nhận request, kiểm tra vai trò, gọi service, ánh xạ kết quả sang mã trạng thái. Không chứa logic nghiệp vụ, không mở transaction.
- **Service** điều phối: nạp dữ liệu, kiểm tra quyền sở hữu, gọi `Domain` để tính toán và kiểm tra điều kiện chuyển trạng thái, mở transaction, ghi nhật ký, tạo thông báo.
- **Domain** giữ công thức và quy tắc: tính tiền điện, tiền phòng theo tỷ lệ ngày ở, số dư thanh lý, điều kiện để hợp đồng được kích hoạt.

Service dùng thẳng `DbContext`. Tác vụ định kỳ gọi cùng service đó, nên logic không bị chép lại ở hai nơi.

### 2.3 Mức độ phức tạp cho phép

Phase 1 **không dùng**: repository pattern, CQRS, MediatR, generic service base class, event bus, cache layer.

Riêng về repository: `DbSet<T>` của EF Core đã đóng vai trò repository và `DbContext` đã là Unit of Work, nên thêm một lớp nữa là bọc lại thứ đã được bọc. Hai chỗ trong hệ thống này sẽ vướng ngay nếu có repository — bộ lọc tìm kiếm phòng với nhiều tham số tùy chọn, và chuỗi kiểm tra sở hữu hóa đơn → hợp đồng → phòng → khu trọ → chủ trọ. Cả hai cần ghép truy vấn linh hoạt, thứ mà repository hoặc làm mất đi, hoặc phải đánh đổi bằng rất nhiều method gần giống nhau.

Chỉ thêm một thành phần kiến trúc mới khi có một yêu cầu cụ thể đòi hỏi nó, không thêm vì đó là thực hành phổ biến.

---

## 3. Luồng xử lý một request

Lấy `POST /api/v1/invoices/{id}/payment-reports/{reportId}/confirm` làm ví dụ — Chủ trọ xác nhận đã thu tiền:

| Bước | Tầng | Việc phải làm | Sai thì hậu quả |
|---|---|---|---|
| 1 | Api — middleware | Xác thực JWT, lấy `userId` và vai trò **từ token** | Giả mạo danh tính |
| 2 | Api — middleware | Rate limiting theo nhóm endpoint. Đứng sau bước 1 vì có nhóm tính ngưỡng theo tài khoản đăng nhập | Bị dò mật khẩu, spam |
| 3 | Api — controller | Kiểm tra vai trò: phải là Chủ trọ | Leo thang đặc quyền |
| 4 | Api — controller | Kiểm tra dữ liệu vào: số tiền không âm, các trường bắt buộc có mặt; rồi gọi `InvoiceService` | Dữ liệu rác vào database |
| 5 | Api — service | Nạp hóa đơn kèm chuỗi sở hữu: hóa đơn → hợp đồng → phòng → khu trọ → chủ trọ | |
| 6 | Api — service | **Kiểm tra quyền sở hữu**: khu trọ có thuộc người gọi không; không thuộc thì trả `404` | Xem và sửa dữ liệu người khác |
| 7 | Domain | Kiểm tra trạng thái nghiệp vụ: hóa đơn có đang chờ xác nhận không; số tiền so với tổng hóa đơn để quyết định trạng thái kết quả | Bỏ qua bước trong vòng đời, đánh dấu đã trả khi chưa đủ tiền |
| 8 | Api — service | Cập nhật hóa đơn, ghi `audit_logs`, tạo thông báo — trong **một transaction** | Ghi một nửa, mất dấu vết |
| 9 | Database | Ràng buộc `CHECK`, `UNIQUE`, khóa ngoại | Lớp chặn cuối khi 7 tầng trên sót |

**Năm lớp kiểm soát độc lập**, khớp với [Thiết kế An toàn](security-design.md) mục 7: bước 1 (xác thực — ai), bước 3 và 6 (phân quyền — vai trò nào, tài nguyên của ai), bước 4 (dữ liệu vào có đúng định dạng không), bước 7 (nghiệp vụ có cho phép không), bước 9 (dữ liệu lưu có hợp lệ không). Không lớp nào được bỏ vì "lớp kia kiểm rồi".

---

## 4. Logic nào đặt ở đâu

| Loại logic | Đặt ở | Ví dụ |
|---|---|---|
| Kiểm tra định dạng, trường bắt buộc, khoảng giá trị | Api (FluentValidation) | Email đúng định dạng, số tiền ≥ 0, ngày kết thúc sau ngày bắt đầu |
| Ánh xạ entity sang dữ liệu trả về | Api — `Contracts/` | DTO của hóa đơn chỉ gồm trường người thuê được xem |
| Kiểm tra vai trò | Api — controller | Endpoint này chỉ dành cho Chủ trọ |
| Kiểm tra quyền sở hữu | Api — service | Khu trọ này có thuộc người gọi không |
| Điều phối nhiều bước, mở transaction | Api — service | Duyệt yêu cầu thuê: giữ chỗ phòng, từ chối các yêu cầu còn lại, thông báo, ghi nhật ký |
| Điều kiện chuyển trạng thái | Domain | Hợp đồng chỉ sang Đang hiệu lực khi đã xác nhận điều khoản và đã nhận cọc |
| Công thức tính tiền | Domain | Tiền điện, tiền phòng theo tỷ lệ ngày ở, tổng hóa đơn, số dư thanh lý |
| Quy tắc nghiệp vụ liên quan nhiều thực thể | Domain | Một phòng chỉ có một hợp đồng đang chiếm dụng |
| Ánh xạ entity sang bảng, migration, ràng buộc | Infrastructure | Đặt tên `snake_case`, ràng buộc `CHECK` chỉ số, unique index |
| Cách lưu nhật ký và thông báo | Infrastructure | Bảng `audit_logs`, `notifications` |
| Quyết định **khi nào** ghi nhật ký và gửi thông báo | Api — service | Ghi nhật ký ngay sau khi xác nhận thanh toán, trong cùng transaction |
| Ràng buộc dữ liệu không được phá vỡ trong mọi hoàn cảnh | Database | Chỉ số mới ≥ chỉ số cũ, một hóa đơn định kỳ cho mỗi kỳ |

**Nguyên tắc chọn tầng:** nếu quy tắc đó vẫn đúng khi hệ thống đổi sang database khác hoặc đổi sang giao diện khác, nó thuộc về `Domain`. Nếu nó chỉ tồn tại vì cách dữ liệu được lưu, nó thuộc `Infrastructure`. Nếu nó chỉ tồn tại vì cách client gửi dữ liệu lên, nó thuộc `Api`.

---

## 5. Dịch vụ ngoài

Hai dịch vụ ngoài được gọi từ `Infrastructure`, mỗi cái nằm sau một interface để tầng service không phụ thuộc vào chi tiết kết nối:

| Dịch vụ | Cách gọi | Dùng cho |
|---|---|---|
| **Supabase Storage** | `HttpClient` gọi thẳng REST API, không dùng thư viện ngoài | Tải file lên, tạo URL có chữ ký cho file riêng tư, xoá file |
| **SMTP Gmail** | MailKit | Gửi email chứa đường dẫn đặt lại mật khẩu |

Lý do không dùng thư viện client của Supabase: hệ thống chỉ cần ba thao tác với Storage, trong khi .NET 10 còn mới và thư viện cộng đồng có thể chưa kịp hỗ trợ. Gọi thẳng REST giữ được quyền kiểm soát và không thêm rủi ro tương thích.

Khi dịch vụ ngoài lỗi, thao tác nghiệp vụ tương ứng **thất bại rõ ràng** chứ không âm thầm bỏ qua — không lưu được ảnh giấy tờ thì hồ sơ không được tạo.

---

## 6. Cấu hình và secret

Chuỗi kết nối Supabase, khóa ký JWT, khóa truy cập Supabase Storage và mật khẩu ứng dụng SMTP **không nằm trong file cấu hình được commit**. Khi phát triển, các giá trị này đọc từ User Secrets của .NET; khi chạy thật, đọc từ biến môi trường. Repository chỉ chứa file cấu hình mẫu với giá trị rỗng.

Chi tiết lý do và các quy tắc an toàn khác: [Thiết kế An toàn](security-design.md).

---

## 7. Tác vụ định kỳ

Năm hành vi của hệ thống xảy ra theo thời gian chứ không do người dùng kích hoạt:

| Tác vụ | Kết quả |
|---|---|
| Hết hạn yêu cầu thuê: quá 168 giờ (7 ngày) kể từ lúc gửi mà chưa xử lý | Yêu cầu chuyển sang Hết hạn |
| Nhắc nộp cọc: hạn giữ chỗ còn dưới 24 giờ mà hợp đồng chưa có hiệu lực | Người thuê nhận một thông báo nhắc |
| Hết hạn giữ chỗ: quá 72 giờ (3 ngày) kể từ khi duyệt yêu cầu thuê mà hợp đồng chưa có hiệu lực | Hợp đồng (nếu đã lập) chuyển Đã hủy, yêu cầu thuê chưa lập hợp đồng chuyển Hết hạn, phòng trở lại Trống |
| Gắn cờ quá hạn: hóa đơn chưa trả đủ đã sang ngày sau hạn thanh toán | Hóa đơn chuyển Quá hạn, gửi thông báo cho hai bên |
| Đánh dấu hợp đồng còn 15 ngày tới ngày kết thúc | Hợp đồng chuyển Sắp hết hạn, gửi thông báo |

Các tác vụ chạy **mỗi giờ** trong tiến trình của Backend API, không tách thành dịch vụ riêng; một lần chạy xử lý cả năm loại, nên kết quả trễ tối đa một giờ. Mỗi tác vụ phải chạy lại được nhiều lần mà không gây tác dụng phụ lặp lại — ví dụ trước khi gửi lời nhắc nộp cọc, kiểm tra đã có thông báo cùng loại cho hợp đồng đó chưa.

### 7.1 Quy ước thời gian

Thời điểm được lưu theo UTC (`timestamptz`). Mọi "ngày" nghiệp vụ — ngày bắt đầu và kết thúc hợp đồng, hạn thanh toán, ngày trả phòng, "hôm nay" — tính theo giờ Việt Nam (`Asia/Ho_Chi_Minh`):

- Hóa đơn quá hạn từ 0 giờ của ngày sau `due_date`.
- Hạn tính bằng ngày nhưng bắt đầu từ một thời điểm thì tính đủ giờ: hạn xử lý yêu cầu thuê 7 ngày là 168 giờ kể từ lúc gửi, hạn giữ chỗ 3 ngày là 72 giờ kể từ lúc duyệt.

---

## 8. Frontend

| Thành phần | Trách nhiệm |
|---|---|
| React + TypeScript | Giao diện |
| React Router | Định tuyến theo vai trò; mỗi màn hình có URL riêng |
| Tailwind CSS | Trình bày; giao diện phải dùng được trên màn hình rộng từ 360px |
| React Hook Form | Nhập liệu và kiểm tra dữ liệu ở mức trải nghiệm người dùng |
| TanStack Query | Gọi API, lưu tạm dữ liệu, đồng bộ lại sau khi thay đổi |
| Axios | Gắn JWT vào header cho mọi request; token đọc từ `localStorage` |
| qrcode | Vẽ mã VietQR ngay trong trình duyệt từ trường `paymentQr` do backend trả về (BR-26); không gọi dịch vụ tạo ảnh QR bên ngoài |

**Kiểm tra dữ liệu ở frontend là để người dùng đỡ phải gửi request sai, không phải để bảo vệ hệ thống.** Ẩn nút hay chặn route theo vai trò cũng vậy. Mọi kiểm soát thật nằm ở backend và được thực hiện lại cho từng request.
