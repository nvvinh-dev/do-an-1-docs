# Thiết kế An toàn — Phase 1

Tài liệu chính thức về security của hệ thống SmartRent.

**Phạm vi:** Phase 1 theo phân kỳ trong [README](../README.md). Tài liệu này xác định tài sản cần bảo vệ, ma trận phân quyền, mô hình mối đe dọa và các quy tắc thiết kế an toàn bắt buộc phải tuân thủ khi hiện thực.

Tài liệu này đi cùng [Thiết kế Cơ sở dữ liệu](database-design.md) và [Thiết kế API](api-design.md). Khi một quy tắc ở đây mâu thuẫn với hai tài liệu kia, tài liệu này là căn cứ cho phần security.

---

## 1. Tài sản cần bảo vệ

| Tài sản | Nội dung cụ thể trong SmartRent |
|---|---|
| **Tài khoản** | Email, mật khẩu, vai trò của Admin, Chủ trọ, Người thuê |
| **Dữ liệu nhân thân** | Số CCCD, ảnh hai mặt CCCD, giấy tờ chứng minh quyền sở hữu bất động sản |
| **Thông tin liên hệ** | Số điện thoại của Chủ trọ và Người thuê |
| **Tài khoản nhận tiền** | Ngân hàng, số tài khoản và tên chủ tài khoản của Chủ trọ |
| **Hợp đồng** | Điều khoản, giá đã chốt, số tiền cọc |
| **Hóa đơn và thanh toán** | Chỉ số điện nước, đơn giá, số tiền, minh chứng thanh toán |
| **Tiền cọc** | Số tiền cọc, toàn bộ khoản khấu trừ khi thanh lý, và phần cọc giữ lại khi Người thuê hủy hợp đồng |
| **Doanh thu** | Số liệu doanh thu của từng Chủ trọ |
| **Nhật ký hệ thống** | Bản ghi đối chứng khi xảy ra tranh chấp |

### 1.1 Phân tích theo Confidentiality — Integrity — Availability

| Khía cạnh | Yêu cầu trong SmartRent |
|---|---|
| **Confidentiality** — ai được xem | Người thuê không được xem hợp đồng và hóa đơn của người thuê khác. Chủ trọ không được xem dữ liệu của khu trọ không thuộc mình. Ảnh CCCD chỉ Admin xem trong lúc duyệt hồ sơ. Số điện thoại chỉ lộ khi đã có cơ sở nghiệp vụ |
| **Integrity** — ai được sửa | Chỉ số điện nước, đơn giá và số tiền chỉ được thay đổi bởi Chủ trọ sở hữu, trong trạng thái cho phép, và mọi thay đổi đều để lại dấu vết. Hóa đơn đã thanh toán không ai sửa được, kể cả Admin. Nhật ký hệ thống không ai sửa được |
| **Availability** — hệ thống sẵn sàng | Người thuê phải xem được hóa đơn và hợp đồng của mình vào thời điểm đến hạn thanh toán. Chủ trọ phải chốt được chỉ số vào kỳ hóa đơn |

---

## 2. Xác thực và phân quyền

**Authentication — bạn là ai.** Mọi vai trò đều đăng nhập bằng email và mật khẩu, nhận JWT. Mật khẩu do ASP.NET Identity băm, không bao giờ lưu dạng gốc và không bao giờ xuất hiện trong bất kỳ response nào.

**Vòng đời token.** Access token sống 60 phút, hết hạn thì đăng nhập lại; hệ thống không dùng refresh token. Frontend lưu token trong `localStorage` để giữ phiên qua các lần tải lại trang.

Cách lưu này có một hệ quả phải chấp nhận: nếu trang có lỗ hổng XSS thì mã độc đọc được token. Vì vậy frontend **không bao giờ** dựng HTML từ dữ liệu người dùng nhập (không dùng `dangerouslySetInnerHTML` với nội dung đến từ API). React tự escape khi render bình thường — giữ nguyên cách đó là đủ.

Một hệ quả thứ hai: khóa tài khoản không cắt được phiên đang mở, người bị khóa vẫn thao tác được cho tới khi token hết hạn. Với thời hạn 60 phút, đây là rủi ro nhóm chấp nhận để đổi lấy việc không phải xây cơ chế thu hồi token.

**Authorization — bạn được phép làm gì.** Quyền được quyết định ở backend dựa trên bốn yếu tố: **người dùng + hành động + tài nguyên + ngữ cảnh**. Biết vai trò là chưa đủ — phải kiểm tra cả quyền sở hữu tài nguyên và trạng thái nghiệp vụ hiện tại của nó.

### 2.1 Ma trận phân quyền

`✓` được phép · `✗` không được phép · `Sở hữu` chỉ trên tài nguyên thuộc về mình · `—` không áp dụng

| Chức năng | Khách | Người thuê | Chủ trọ | Admin |
|---|---|---|---|---|
| Tìm kiếm và xem phòng công khai | ✓ | ✓ | ✓ | ✓ |
| Đăng ký, đăng nhập, quên mật khẩu | ✓ | — | — | — |
| Đổi mật khẩu, cập nhật thông tin cá nhân | ✗ | ✓ | ✓ | ✓ |
| Nộp hồ sơ đăng ký Chủ trọ | ✗ | ✓ | ✗ | ✗ |
| Xem ảnh CCCD trong hồ sơ | ✗ | ✗ | ✗ | ✓ |
| Duyệt / từ chối hồ sơ Chủ trọ | ✗ | ✗ | ✗ | ✓ |
| Khóa / mở khóa tài khoản | ✗ | ✗ | ✗ | ✓ |
| Tạo, sửa, lưu trữ khu trọ và phòng | ✗ | ✗ | Sở hữu | ✗ |
| Bật / tắt hiển thị tin | ✗ | ✗ | Sở hữu | ✗ |
| Gửi / hủy yêu cầu thuê | ✗ | ✓ | ✗ | ✗ |
| Duyệt / từ chối yêu cầu thuê | ✗ | ✗ | Sở hữu | ✗ |
| Lập và sửa hợp đồng | ✗ | ✗ | Sở hữu | ✗ |
| Xác nhận điều khoản hợp đồng | ✗ | Đứng tên | ✗ | ✗ |
| Xác nhận đã nhận tiền cọc | ✗ | ✗ | Sở hữu | ✗ |
| Hủy hợp đồng trước ngày bắt đầu | ✗ | Đứng tên | Sở hữu | ✗ |
| Ghi nhận đã hoàn cọc | ✗ | ✗ | Sở hữu | ✗ |
| Gửi thông báo trả phòng | ✗ | Đứng tên | Sở hữu | ✗ |
| Rút thông báo trả phòng | ✗ | Bên đã gửi | Bên đã gửi | ✗ |
| Khai báo tài khoản ngân hàng nhận tiền | ✗ | ✗ | Của mình | ✗ |
| Xem mã VietQR để chuyển khoản | ✗ | Đứng tên | ✗ | ✗ |
| Chốt chỉ số, tạo và phát hành hóa đơn | ✗ | ✗ | Sở hữu | ✗ |
| Báo đã thanh toán kèm minh chứng | ✗ | Đứng tên | ✗ | ✗ |
| **Xác nhận đã thu tiền** | ✗ | ✗ | **Sở hữu** | **✗** |
| Lập, gửi, tự chốt hóa đơn thanh lý; tất toán cọc | ✗ | ✗ | Sở hữu | ✗ |
| Đồng ý / chưa đồng ý bảng thanh lý | ✗ | Đứng tên | ✗ | ✗ |
| Xem hợp đồng và hóa đơn | ✗ | Của mình | Sở hữu | ✗ |
| Xem dashboard | ✗ | Của mình | Của mình | Toàn hệ thống |
| Tra cứu nhật ký hệ thống | ✗ | ✗ | ✗ | ✓ |

**Hai ô cần chú ý:**

- **Admin không xác nhận thanh toán.** Chỉ Chủ trọ sở hữu mới xác nhận được một hóa đơn đã thu đủ. Admin có quyền cao nhất về quản trị nhưng không có quyền động vào tiền của người khác.
- **Admin không đọc hợp đồng và hóa đơn.** Phase 1 không có endpoint nào cho phép việc này. Quyền đọc chỉ mở khi xử lý khiếu nại, và cơ chế đó thuộc Phase 2.

Ẩn tin vi phạm theo từng phòng cũng thuộc Phase 2. Phase 1 Admin xử lý vi phạm bằng khóa tài khoản Chủ trọ.

---

## 3. Ranh giới tin cậy

```
[ Trình duyệt — KHÔNG đáng tin ]
   người dùng sửa được request, token, DevTools, Postman
                │
                ▼  ← ranh giới tin cậy
[ Backend API — nơi thực thi mọi kiểm soát ]
   authentication → authorization → validation → business rules
                │
                ▼
[ PostgreSQL — chỉ nhận dữ liệu đã kiểm tra ]
   ràng buộc CHECK, UNIQUE, khóa ngoại là lớp phòng vệ cuối
```

Frontend chạy trong vùng không đáng tin. Ẩn nút, disable ô nhập, chặn route ở frontend là trải nghiệm người dùng, **không phải** kiểm soát an toàn. Mọi request tới backend đều được xác thực và phân quyền lại, bất kể đến từ đâu.

**Chỉ có một đường tới dữ liệu.** Supabase mặc định phơi bày mọi bảng trong schema `public` qua REST API riêng của nó, và anon key vốn được thiết kế để nhúng vào client. Nếu để nguyên, người dùng có thể đọc ghi thẳng vào bảng và đi vòng qua toàn bộ kiểm soát ở backend.

Vì vậy **Data API của Supabase được tắt** — bỏ schema `public` khỏi danh sách Exposed schemas. Hệ thống dùng Supabase như một PostgreSQL thông thường qua chuỗi kết nối, chỉ backend truy cập được. Ai thêm bảng mới cũng không cần làm gì thêm, vì cả schema đã không còn được phơi bày.

---

## 4. Mô hình mối đe dọa

| Loại | Kịch bản cụ thể trong SmartRent | Kiểm soát |
|---|---|---|
| **Spoofing**<br>giả mạo danh tính | Kẻ tấn công dùng lại token của người thuê khác, hoặc đăng nhập bằng mật khẩu đoán được | Xác thực bằng JWT có hạn; mật khẩu băm bằng ASP.NET Identity; Admin khóa được tài khoản và tài khoản bị khóa không đăng nhập được |
| **Tampering**<br>sửa dữ liệu trái phép | Người thuê sửa `totalAmount` trong request để giảm số tiền phải trả; Chủ trọ gửi đơn giá điện cao hơn đơn giá đã chốt trong hợp đồng | Server tự tính toàn bộ số tiền từ đơn giá lưu trong hợp đồng; client không được gửi số tiền lên. Ràng buộc `CHECK` chặn chỉ số mới nhỏ hơn chỉ số cũ |
| **Repudiation**<br>chối bỏ hành vi | Chủ trọ sửa chỉ số điện rồi phủ nhận; Admin thu hồi vai trò rồi nói không làm; Chủ trọ đổi số tài khoản nhận tiền rồi nói người thuê chuyển nhầm | `audit_logs` ghi người thực hiện, thời điểm, giá trị trước và sau cho mọi thao tác ảnh hưởng tới tiền hoặc quyền. Bảng này chỉ cho phép INSERT |
| **Information Disclosure**<br>lộ thông tin | Người thuê đổi id trên URL để xem hợp đồng của người khác; ảnh CCCD bị lộ; số điện thoại Chủ trọ bị thu thập hàng loạt từ kết quả tìm kiếm | Kiểm tra quyền sở hữu trên mọi tài nguyên; trả `404` thay vì `403` với tài nguyên có thể bị dò id; ảnh giấy tờ chỉ trả cho Admin khi duyệt hồ sơ; kết quả tìm kiếm không chứa thông tin liên hệ |
| **Denial of Service**<br>làm hệ thống ngừng phục vụ | Thử mật khẩu hàng loạt để chiếm tài khoản; gửi liên tục yêu cầu thuê; cào toàn bộ dữ liệu phòng | Rate limiting theo chính sách ở Mục 6; Admin khóa tài khoản lạm dụng; một người thuê không tạo được nhiều yêu cầu đang chờ cho cùng một phòng (BR-27) |
| **Elevation of Privilege**<br>leo thang đặc quyền | Người dùng gửi `"role": "Landlord"` trong body để tự nâng quyền; gọi thẳng endpoint của Admin | Vai trò **chỉ** đọc từ token do server cấp, không bao giờ đọc từ body hay query. Vai trò Chủ trọ chỉ được cấp qua luồng Admin duyệt hồ sơ |

---

## 5. Năm thiết kế không an toàn — đối chiếu với SmartRent

### 5.1 Broken Access Control

**Rủi ro trong hệ thống này:** `GET /api/v1/invoices/12` — nếu chỉ tìm hóa đơn theo id rồi trả về, bất kỳ ai đăng nhập cũng xem được hóa đơn của người khác, kèm toàn bộ số tiền và chỉ số điện nước.

**Quy tắc:** mọi endpoint thao tác trên một tài nguyên cụ thể phải kiểm tra theo thứ tự: xác thực người dùng → tìm tài nguyên → **kiểm tra quyền sở hữu hoặc vai trò** → kiểm tra trạng thái nghiệp vụ → mới thực hiện. Không được rút gọn bước ba.

Quyền sở hữu trong hệ thống này: hóa đơn thuộc hợp đồng, hợp đồng thuộc phòng, phòng thuộc khu trọ, khu trọ thuộc Chủ trọ. Kiểm tra phải truy ngược đủ chuỗi này, không dừng ở việc "người gọi có vai trò Chủ trọ".

### 5.2 Trusting Client Input

**Rủi ro trong hệ thống này:**

| Giá trị | Vì sao không được tin client |
|---|---|
| `userId` của người thao tác | Gửi id người khác để thao tác dưới danh nghĩa họ |
| Vai trò | Tự nâng mình thành Chủ trọ hoặc Admin |
| Chỉ số điện nước cũ | Khai chỉ số cũ cao lên để giảm số tiền điện |
| Mọi số tiền trên hóa đơn | Tự đặt tổng tiền bằng 0 |
| Số tiền trong mã VietQR | Hiển thị số tiền sai lệch với khoản thực sự phải trả |
| Số tiền trừ cọc và số tiền hoàn cọc | Trừ hoặc hoàn khác với số cọc thật ghi trong hợp đồng |
| Đường dẫn file gắn vào request nghiệp vụ | Gắn file của người khác, hoặc file sai loại, vào hồ sơ của mình |
| Đơn giá điện, nước, giá thuê khi lập hóa đơn | Bỏ qua giá đã chốt trong hợp đồng |
| Ngày đầu và ngày cuối kỳ hóa đơn | Lập chồng kỳ để thu tiền phòng hai lần |

**Quy tắc:** danh tính và vai trò của người thao tác **luôn lấy từ token**. Chỉ số cũ **luôn lấy từ hóa đơn kỳ liền trước**, kỳ đầu tiên lấy chỉ số đầu ghi trong hợp đồng. Đơn giá **luôn lấy từ hợp đồng**. Mọi số tiền **do server tính**. Client gửi các giá trị này lên thì bỏ qua, không dùng.

**Ngoại lệ duy nhất về số tiền:** khi Người thuê hủy hợp đồng trước ngày bắt đầu, số tiền hoàn cọc do Chủ trọ nhập (BR-22). Server chỉ nhận giá trị trong khoảng 0 tới tiền cọc ghi trong hợp đồng, bắt buộc kèm lý do khi số hoàn nhỏ hơn tiền cọc, và ghi nhật ký.

### 5.3 Hardcoded Secret

**Quy tắc bắt buộc:** chuỗi kết nối Supabase, khóa truy cập Supabase Storage, khóa ký JWT và mật khẩu ứng dụng SMTP **không được nằm trong source code hay trong file cấu hình được commit**. Khi phát triển, các giá trị này đọc từ User Secrets của .NET; khi chạy thật, đọc từ biến môi trường. Repository chỉ chứa file cấu hình mẫu với giá trị rỗng.

Nếu một secret đã lỡ bị commit, việc xóa dòng đó ở commit sau **không** giải quyết vấn đề — secret vẫn nằm trong lịch sử Git và phải được thay mới.

### 5.4 Excessive Data Exposure

**Rủi ro trong hệ thống này:** bảng `users` chứa `password_hash`; `landlord_applications` chứa số CCCD và ảnh giấy tờ; `contracts` chứa toàn bộ điều khoản tài chính. Trả thẳng dữ liệu bảng ra ngoài là lộ những trường này.

**Quy tắc:** mọi response đi qua một lớp **DTO riêng** đặt trong `SmartRent.Api/Contracts/`. Controller không bao giờ serialize thẳng entity của EF Core — đó là cách các trường nhạy cảm lọt ra ngoài mà không ai nhận ra, nhất là khi entity được thêm cột mới về sau.

Response chỉ chứa các trường mà chức năng đó thực sự cần. Ba nhóm trường **không bao giờ** được xuất hiện trong bất kỳ response nào ngoài đúng ngữ cảnh của chúng:

| Trường | Chỉ được trả khi |
|---|---|
| `password_hash` và mọi trường kỹ thuật của Identity | Không bao giờ |
| Số CCCD, ảnh CCCD, giấy tờ sở hữu | Admin đang xem chi tiết hồ sơ để duyệt, qua URL có chữ ký và có hạn |
| Số điện thoại của bên còn lại | Giữa hai bên có yêu cầu thuê đã được Chủ trọ duyệt, hợp đồng chưa kết thúc, hợp đồng đã hủy còn chờ hoàn cọc, hoặc hợp đồng đã thanh lý mà hóa đơn thanh lý còn nợ (QR-07) |
| Tài khoản ngân hàng của Chủ trọ | Chính Chủ trọ đó xem tài khoản của mình; hoặc Người thuê đứng tên hợp đồng của Chủ trọ, trong trường `paymentQr` khi có khoản cần chuyển khoản (BR-26) |

Kết quả tìm kiếm phòng công khai không chứa thông tin liên hệ hay tài khoản ngân hàng của Chủ trọ.

**Mã VietQR được vẽ ngay trong trình duyệt.** Frontend dựng chuỗi dữ liệu VietQR và vẽ mã QR bằng thư viện cục bộ, không gọi dịch vụ tạo ảnh QR bên ngoài — số tài khoản và số tiền của người dùng không rời khỏi hệ thống.

### 5.5 Lạm dụng logic nghiệp vụ

Các luồng có thể bị lợi dụng trong hệ thống này và cách thiết kế đã chặn:

| Kịch bản lạm dụng | Thiết kế chặn lại |
|---|---|
| Hai yêu cầu thuê cùng một phòng được duyệt gần như đồng thời, tạo hai hợp đồng | Thao tác duyệt chạy trong một transaction, chuyển phòng sang *Đang giữ chỗ* và tự từ chối các yêu cầu còn lại; unique index chặn hai hợp đồng cùng chiếm dụng một phòng |
| Hợp đồng được kích hoạt mà chưa nhận cọc | Chỉ chuyển sang *Đang hiệu lực* khi có đủ cả xác nhận của người thuê lẫn xác nhận đã nhận cọc |
| Người thuê chụp màn hình mã VietQR rồi coi như đã trả tiền | Quét mã hay chuyển khoản không đổi trạng thái hóa đơn; tiền chỉ được ghi nhận khi Chủ trọ xác nhận (BR-06b, BR-26) |
| Chủ trọ đánh dấu đã thu đủ trong khi người thuê mới trả một phần | Số tiền xác nhận được so với tổng hóa đơn; thiếu thì trạng thái là *Thanh toán một phần*, phần còn lại vẫn là công nợ |
| Chủ trọ sửa chỉ số sau khi người thuê đã trả tiền | Hóa đơn ở *Đã thanh toán* không sửa được; sai sót chỉ được điều chỉnh bằng một dòng riêng ở kỳ sau, có mô tả, tham chiếu tới hóa đơn gốc và được ghi nhật ký |
| Khấu trừ hết tiền cọc mà không giải thích | Mỗi khoản khấu trừ trong hóa đơn thanh lý bắt buộc là một dòng riêng có mô tả lý do; khi Người thuê hủy trước ngày bắt đầu, phần cọc Chủ trọ giữ lại bắt buộc kèm lý do và được ghi nhật ký |
| Chủ trọ tự chấm dứt hợp đồng rồi vẫn thu phí phạt của người thuê | Hệ thống ghi nhận bên gửi thông báo trả phòng; dòng phí phạt chỉ hợp lệ khi người thuê là bên gửi và báo trước dưới 30 ngày (BR-22) |
| Chủ trọ tự chốt bảng thanh lý để ép người thuê chịu các khoản khấu trừ | Chỉ tự chốt được sau 7 ngày người thuê không phản hồi, bắt buộc ghi chú, ghi nhật ký và thông báo cho người thuê; mọi khoản khấu trừ vẫn là dòng riêng có mô tả. Từ Phase 2 người thuê khiếu nại được |
| Tạo hai hóa đơn cho cùng một kỳ để thu tiền hai lần | Kỳ hóa đơn do server xác định, client không gửi ngày kỳ; unique index trên hợp đồng và kỳ là lớp chặn cuối |
| Xóa phòng để phi tang lịch sử hóa đơn | Không có endpoint xóa; phòng đã phát sinh giao dịch chỉ được chuyển sang *Lưu trữ* |
| Chủ trọ tăng giá giữa chừng để tính lại hóa đơn cũ | Giá được chốt cứng vào hợp đồng; mỗi hóa đơn lưu bản sao đơn giá đã áp dụng |

---

## 6. Chính sách rate limiting

Rate limiting là kiểm soát **bắt buộc**, không phải tùy chọn. Không thêm thư viện ngoài: các nhóm tính theo số lần gọi dùng middleware rate limiting có sẵn của ASP.NET Core; riêng nhóm đăng nhập chỉ đếm lần sai nên dùng bộ đếm riêng trong bộ nhớ của tiến trình (`LoginAttemptLimiter`), bộ đếm mất khi ứng dụng khởi động lại. Hệ thống không dùng cơ chế khóa tạm theo tài khoản của ASP.NET Identity, vì cơ chế đó cho phép bất kỳ ai biết email khóa được tài khoản của người khác.

| Nhóm endpoint | Giới hạn | Tính theo |
|---|---|---|
| Đăng nhập | 5 lần sai liên tiếp mỗi 15 phút | Email + địa chỉ IP |
| Đăng ký tài khoản, quên mật khẩu, đặt lại mật khẩu | 3 lần mỗi giờ | Địa chỉ IP |
| Gửi yêu cầu thuê, báo đã thanh toán, nộp hồ sơ Chủ trọ | 10 lần mỗi giờ | Tài khoản đăng nhập |
| Tải file | 20 lần mỗi giờ | Tài khoản đăng nhập |
| Tìm kiếm phòng công khai | 60 lần mỗi phút | Địa chỉ IP |

Các nhóm tính theo tài khoản đăng nhập đọc id người dùng từ token, nên middleware rate limiting chạy sau bước xác thực JWT.

**Mỗi endpoint đếm riêng.** Ngưỡng trong bảng áp dụng cho từng endpoint của nhóm, không cộng dồn — đăng ký và đặt lại mật khẩu không dùng chung 3 lượt. Ngưỡng đọc từ cấu hình: bảng trên là giá trị cho môi trường thật, môi trường Development được nới để kiểm thử bằng Postman.

**Hành vi khi vượt ngưỡng:** trả `429 Too Many Requests` kèm header `Retry-After`, thân phản hồi dạng ProblemDetails như mọi lỗi khác. Lượt vượt ngưỡng ở nhóm đăng nhập được ghi log để Admin đối chiếu khi nghi ngờ tài khoản bị tấn công.

**Đếm theo lần sai, không theo lần gọi.** Ở nhóm đăng nhập, chỉ những lần đăng nhập **thất bại** mới tính vào ngưỡng. Đăng nhập thành công không làm người dùng thật cạn lượt.

**Thông báo lỗi không tiết lộ thông tin.** Phản hồi khi vượt ngưỡng không được cho biết email đó có tồn tại trong hệ thống hay không — email không tồn tại cũng bị đếm và bị chặn như mọi email khác. Lý do khóa tài khoản chỉ hiện ra khi email và mật khẩu đã đúng.

---

## 7. Sáu nguyên tắc áp dụng vào SmartRent

| Nguyên tắc | Áp dụng cụ thể |
|---|---|
| **Least Privilege** | Người thuê chỉ thấy dữ liệu của chính mình; Chủ trọ chỉ thao tác trên khu trọ của mình; Admin quản trị được tài khoản nhưng không đọc hợp đồng, hóa đơn và không xác nhận thanh toán |
| **Defense in Depth** | Năm lớp độc lập: xác thực → phân quyền → kiểm tra dữ liệu đầu vào → quy tắc nghiệp vụ → ràng buộc database. Ràng buộc `CHECK` chỉ số và unique index tồn tại ngay cả khi tầng ứng dụng sót |
| **Secure by Default** | Tài khoản mới mặc định là Người thuê, không phải Chủ trọ. Phòng mới mặc định chưa hiển thị (Đã ẩn bởi Chủ trọ) cho tới khi Chủ trọ chủ động bật |
| **Fail Securely** | Khi không xác định được quyền — thiếu token, token hỏng, lỗi khi truy vấn quyền sở hữu — hệ thống **từ chối** request và ghi log. Không bao giờ cho qua vì "chưa chắc là sai" |
| **Minimize Attack Surface** | Chỉ mở endpoint mà Phase 1 thực sự cần. Không có endpoint xóa dữ liệu tài chính, không có endpoint sửa nhật ký, không có endpoint cho Admin đọc hợp đồng. **Swagger chỉ bật ở môi trường Development**, tắt hoàn toàn ở môi trường thật |
| **Complete Mediation** | Backend kiểm tra lại quyền ở **mọi** request, không tin rằng frontend đã kiểm tra. Áp dụng như nhau cho request từ web, Postman hay bất kỳ nguồn nào |

---

## 8. Quy tắc bắt buộc khi hiện thực

Danh sách này được dùng làm checklist khi review code:

1. Danh tính và vai trò của người thao tác lấy từ token, không bao giờ từ body hoặc query.
2. Mọi endpoint thao tác trên tài nguyên cụ thể đều kiểm tra quyền sở hữu trước khi thực hiện.
3. Tài nguyên có thể bị dò id — hợp đồng, hóa đơn, hồ sơ Chủ trọ — trả `404` khi người gọi không có quyền, không trả `403`.
4. Mọi số tiền và chỉ số cũ do server tính hoặc tra ra, không nhận từ client — kể cả số tiền trong mã VietQR. Ngoại lệ duy nhất là số tiền hoàn cọc khi Người thuê hủy hợp đồng, xem 5.2.
5. Response không chứa `password_hash`, số CCCD, ảnh giấy tờ, tài khoản ngân hàng của Chủ trọ, ngoài đúng ngữ cảnh đã nêu ở 5.4.
6. Secret đọc từ User Secrets hoặc biến môi trường, không nằm trong file được commit.
7. Mọi thao tác thuộc danh sách của BR-23 đều ghi `audit_logs` với giá trị trước và sau.
8. Khi không xác định được quyền, từ chối request.
9. Thao tác đổi trạng thái chỉ chấp nhận các chuyển tiếp đã khai báo trong vòng đời tương ứng.
10. Mọi thao tác đổi trạng thái nhiều bản ghi cùng lúc chạy trong một transaction: duyệt yêu cầu thuê, rút yêu cầu đã duyệt, tạo và hủy hợp đồng, lập hóa đơn thanh lý (kết chuyển công nợ), hoàn tất thanh lý, lưu trữ khu trọ.
11. Các nhóm endpoint ở Mục 6 đều được gắn rate limiting đúng ngưỡng đã quy định.
12. Response trả về là DTO trong `Contracts/`, không phải entity của EF Core.
13. File nhạy cảm nằm ở bucket riêng tư và chỉ truy cập qua URL có chữ ký, có hạn.
14. Swagger chỉ được bật khi môi trường là Development.
15. Frontend không dựng HTML từ dữ liệu người dùng nhập — token nằm trong `localStorage` nên một lỗ hổng XSS là mất token.
16. Data API của Supabase luôn ở trạng thái tắt; dữ liệu chỉ đến được qua backend.
17. Đường dẫn file gắn vào request nghiệp vụ phải đúng bucket, đúng thư mục của `purpose` và nằm trong thư mục của chính người gọi.
