# Kế hoạch triển khai

Kế hoạch phân chia công việc và thời hạn cho hai thành viên.

**Quỹ thời gian:** 07/09/2026 → 03/12/2026. Phân tích, thiết kế, hạ tầng và BP-01 xong trước 01/10; phần cài đặt còn lại từ 01/10 đến 03/12, khoảng 9 tuần.
**Phân bổ:** Phase 1 từ 01/10 đến 15/11, Phase 2 từ 16/11 đến 29/11, tổng kết từ 30/11 đến 03/12.

---

## 1. Nguyên tắc phân chia

Mỗi người **sở hữu trọn một mảng từ backend tới frontend**. Không ai phải chờ người kia định nghĩa contract, và khi ghép chỉ cần khớp ở vài điểm chung.

| Thành viên | Mảng phụ trách |
|---|---|
| **Giang** | Tài khoản, hồ sơ Chủ trọ, yêu cầu thuê, hợp đồng, thanh lý, thông báo, nhật ký, khu vực Admin |
| **Vinh** | Khu trọ, phòng, tin đăng, tìm kiếm, hóa đơn, thanh toán, khu vực Chủ trọ |

---

## 2. Đã hoàn thành

| Hạng mục | Thời gian | Nội dung |
|---|---|---|
| Phân tích nghiệp vụ | 07/09 – 11/09 | Phạm vi, vai trò người dùng, quy trình và quy tắc nghiệp vụ, phân kỳ Phase 1, 2, 3, phân công mảng phụ trách |
| Tài liệu thiết kế | 12/09 – 15/09 | Đặc tả yêu cầu chức năng, sơ đồ use case, kiến trúc, thiết kế database, thiết kế API, thiết kế an toàn |
| Quy ước làm việc | 12/09 – 15/09 | Công nghệ, cấu trúc thư mục, quy ước Git, hướng dẫn cấu hình và chạy dự án |
| Hạ tầng | 16/09 – 18/09 | Backend 3 project, frontend Vite + React + TypeScript + Tailwind, 18 entity, 25 bảng trên Supabase, 2 bucket Storage, Data API đã tắt, dữ liệu nền (3 vai trò, tài khoản Admin, 16 tiện ích) |
| **BP-01** | 16/09 – 18/09 | 17 endpoint: xác thực, hồ sơ Chủ trọ, quản lý tài khoản, tải file dùng chung |
| Hoàn thiện thiết kế và đề cương | 19/09 – 30/09 | Bổ sung trợ lý AI (Google Gemini), hiển thị mã VietQR, cấu trúc thư mục frontend; viết đề cương chi tiết |

Frontend hiện mới có khung dự án và lớp gọi API. Toàn bộ giao diện là khối việc lớn nhất còn lại của Phase 1.

---

## 3. Phase 1 — hạn chót 15/11

### Giang

| Tuần | Việc | Hạn |
|---|---|---|
| 1 | **BP-06 backend** — yêu cầu thuê, duyệt và từ chối, lập hợp đồng, xác nhận cọc, kích hoạt hợp đồng | 11/10 |
| 2 | **BP-10 backend** — thông báo trả phòng, hóa đơn thanh lý, tất toán cọc · 4 endpoint thông báo · endpoint tra cứu nhật ký | 18/10 |
| 3 | **Khung frontend** — định tuyến theo vai trò, layout, xử lý hết hạn token · màn đăng nhập, đăng ký, quên và đặt lại mật khẩu | 25/10 |
| 4 | Frontend Người thuê — nộp hồ sơ Chủ trọ, gửi yêu cầu thuê, xem hợp đồng của mình | 01/11 |
| 5 | Frontend Admin — duyệt hồ sơ, quản lý tài khoản, tra cứu nhật ký, dashboard Admin | 08/11 |
| 6 | Frontend thanh lý · dashboard Người thuê · ghép nối end-to-end | 15/11 |

**Tác vụ định kỳ phụ trách:** hết hạn yêu cầu thuê quá 7 ngày; hết hạn giữ chỗ quá 3 ngày kể từ khi duyệt yêu cầu thuê; đánh dấu hợp đồng sắp hết hạn trước 15 ngày.

### Vinh

| Tuần | Việc | Hạn |
|---|---|---|
| 1 | **BP-02 và BP-03 backend** — khu trọ, phòng, phí dịch vụ, tiện ích, ảnh, bật tắt hiển thị, lưu trữ | 11/10 |
| 2 | **BP-04 backend** — tìm kiếm bằng bộ lọc, chi tiết phòng công khai · dashboard Chủ trọ | 18/10 |
| 3 | **BP-07 backend** — chốt chỉ số, lập và phát hành hóa đơn, báo và xác nhận thanh toán, hóa đơn điều chỉnh | 25/10 |
| 4 | Frontend Chủ trọ — quản lý khu trọ, quản lý phòng, đăng và ẩn tin, tải ảnh | 01/11 |
| 5 | Frontend tìm phòng và chi tiết phòng · duyệt yêu cầu thuê và lập hợp đồng | 08/11 |
| 6 | Frontend hóa đơn — chốt số, phát hành, xem hóa đơn, báo thanh toán, xác nhận thu · ghép nối | 15/11 |

**Tác vụ định kỳ phụ trách:** gắn cờ hóa đơn quá hạn thanh toán và gửi nhắc nhở.

### Mốc 15/11

Phase 1 chạy trọn vẹn một vòng đời thuê phòng trên dữ liệu thật: đăng ký → tìm phòng → yêu cầu thuê → hợp đồng và cọc → hóa đơn và thanh toán → trả phòng và tất toán. Đạt mốc này mới merge `develop` vào `main` và mới được bắt đầu Phase 2.

Trong tuần 6, cả hai cùng kiểm thử: kiểm thử tự động phần tính hóa đơn và thanh lý, kiểm thử API bằng Postman, chạy thử trọn vòng đời thuê phòng.

---

## 4. Phase 2 — hạn chót 29/11

| Tuần | Giang | Vinh |
|---|---|---|
| 7 · 16–22/11 | **BP-08** — báo cáo sự cố và sửa chữa, backend và frontend | **BP-11** — hồ sơ ở ghép, điểm phù hợp, kết nối hai bên |
| 8 · 23–29/11 | **BP-13** — báo cáo, khiếu nại, xử lý vi phạm | **BP-04 A1** tìm kiếm bằng ngôn ngữ tự nhiên · **BP-12** trợ lý AI tra cứu |

Phần AI là rủi ro lớn nhất của Phase 2. Nếu đến 26/11 chưa chạy được thì cắt bỏ, giữ đúng nguyên tắc dừng: không để phần mở rộng làm hỏng phần lõi đã chạy được.

---

## 5. Tuần cuối — 30/11 đến 03/12

Cả hai cùng làm: sơ đồ tuần tự cho các luồng chính, ERD dạng hình, viết báo cáo, chuẩn bị demo, merge `develop` vào `main`.

BP-05 (đặt lịch xem phòng) và BP-09 (gia hạn hợp đồng) thuộc Phase 3, chỉ làm nếu còn dư thời gian.

---

## 6. Quy tắc phối hợp

**Không chờ nhau khi chưa cần.** BP-06 cần có phòng trong database để kiểm thử, nhưng bảng `rooms` đã tồn tại — chèn vài dòng trực tiếp trên Supabase là kiểm thử được, không phải đợi BP-02 xong.

**Migration phải báo trước.** Hai người dùng chung một database, nên trước khi chạy `dotnet ef migrations add` phải nhắn cho người kia, và người kia phải `git pull` lấy migration mới nhất trước khi đổi schema.

**Duyệt PR trong vòng 24 giờ.** Pull request nằm chờ lâu thì nhánh lệch xa `develop` và lúc merge sẽ xung đột nặng. Mỗi sáng kiểm tra xem có PR nào đang chờ mình duyệt.

**Khi duyệt PR, soi theo checklist ở [Thiết kế An toàn](security-design.md) mục 8.** Bốn chỗ dễ sai nhất: kiểm tra quyền sở hữu tài nguyên, giá trị nào do server tự tính, response có phải DTO riêng không, và các thao tác thuộc BR-23 đã ghi nhật ký chưa.
