# System & Business Analysis
## Hệ thống Quản lý và Cho thuê Phòng trọ

| | |
|---|---|
| **Môn học** | Đồ án 1 |
| **Nhóm thực hiện** | Giang, Vinh (02 thành viên) |
| **Thời gian** | Tối đa 2.5 tháng |
| **Giai đoạn** | Business Analysis (trước Requirements Specification) |

---

## 1. Mục đích và phạm vi tài liệu

### 1.1 Mục đích

Tài liệu này xây dựng **mô hình nghiệp vụ** cho Hệ thống quản lý và cho thuê phòng trọ, làm nền tảng cho giai đoạn *Requirements Specification* tiếp theo. Tài liệu mô tả cách hệ thống vận hành trong thực tế: các actor tham gia, các quy trình nghiệp vụ, khái niệm nghiệp vụ, quy tắc nghiệp vụ, vòng đời trạng thái và ranh giới của hệ thống.

### 1.2 Ranh giới tài liệu

Tài liệu này dừng lại ở **phân tích nghiệp vụ thuần túy**. Mọi nội dung thiết kế kỹ thuật — Sơ đồ Use Case, Sơ đồ tuần tự, Thiết kế cơ sở dữ liệu (ERD), Kiến trúc phần mềm, cơ chế xác thực cụ thể, thiết kế API — **không** thuộc tài liệu này và sẽ được trình bày trong *Tài liệu Đặc tả Yêu cầu & Thiết kế Hệ thống* ở giai đoạn kế tiếp.

> **Ngoại lệ duy nhất:** Mục 15 (Bối cảnh môn học) liệt kê định hướng công nghệ theo yêu cầu của môn học. Đây là ràng buộc đầu vào, không phải kết quả phân tích.

### 1.3 Đối tượng đọc

Giảng viên hướng dẫn Đồ án 1, thành viên nhóm phát triển, và bất kỳ ai tiếp nhận dự án ở giai đoạn sau.

---

## 2. Bối cảnh nghiệp vụ

### 2.1 Hiện trạng — As-Is

Hoạt động cho thuê phòng trọ quy mô nhỏ và vừa tại Việt Nam hiện vận hành gần như hoàn toàn thủ công:

| Hoạt động | Cách làm hiện tại | Vấn đề phát sinh |
|---|---|---|
| Quảng bá phòng trống | Đăng tin trên các nhóm Facebook, dán giấy trước cổng | Tin rời rạc, trùng lặp, không xác minh được thật/giả |
| Tìm phòng | Người thuê lướt nhiều nhóm, gọi điện từng chủ | Tốn thời gian, thông tin giá/diện tích không chuẩn hóa |
| Ghi nhận thỏa thuận thuê | Hợp đồng giấy hoặc thỏa thuận miệng | Thất lạc, không tra cứu được điều khoản khi tranh chấp |
| Chốt điện nước | Chủ trọ đọc đồng hồ, ghi sổ tay hoặc chụp ảnh gửi Zalo | Sai sót khi ghi chép, người thuê không kiểm chứng được chỉ số cũ |
| Lập hóa đơn | Tính tay hoặc Excel, gửi ảnh qua Zalo | Nhầm lẫn đơn giá, mất lịch sử, dễ tranh chấp |
| Thu tiền | Tiền mặt hoặc chuyển khoản, không có chứng từ tập trung | Không đối soát được ai đã đóng, đóng bao nhiêu |
| Tiền cọc | Ghi tay, ghi nhớ | **Nguồn tranh chấp lớn nhất** khi trả phòng |
| Báo sự cố | Nhắn tin Zalo | Chìm trong luồng tin nhắn, không theo dõi được đã xử lý hay chưa |
| Tìm bạn ở ghép | Đăng bài lên nhóm sinh viên | Không có tiêu chí khớp, tỷ lệ ở chung thất bại cao |

**Ba nỗi đau lớn nhất:**

1. **Tranh chấp tiền bạc** do không có dữ liệu đối chứng (chỉ số điện nước, tiền cọc, lịch sử thanh toán).
2. **Rủi ro lừa đảo** do không có cơ chế xác minh chủ trọ và tin đăng.
3. **Mất thời gian** cho các thao tác lặp lại hàng tháng (chốt số, tính tiền, nhắc nợ).

### 2.2 Định hướng — To-Be

Hệ thống phục vụ mô hình kết nối **đa bên** giữa Chủ trọ và Người thuê trọ, dưới sự giám sát của Admin, cung cấp nền tảng số hóa toàn bộ vòng đời thuê trọ: từ tìm kiếm → xem phòng → ký hợp đồng → lưu trú (điện nước, hóa đơn, sự cố) → trả phòng, có Trợ lý AI hỗ trợ tra cứu và kết nối ở ghép.

| Vai trò | Giá trị nhận được |
|---|---|
| **Admin** | Kiểm duyệt các thành phần tham gia (đặc biệt là Chủ trọ), đảm bảo nền tảng minh bạch, xử lý khiếu nại dựa trên dữ liệu đối chứng có thật. |
| **Chủ trọ** | Số hóa công tác quản lý phòng, người thuê, hợp đồng, hóa đơn và doanh thu; giảm thời gian lập hóa đơn hàng tháng. |
| **Người thuê** | Tìm phòng đúng nhu cầu nhanh hơn, theo dõi minh bạch chi phí và hợp đồng, có kênh chính thức để báo sự cố và tìm bạn ở ghép. |
| **AI** | "Trợ lý ảo" hỗ trợ bằng ngôn ngữ tự nhiên trong tìm kiếm, tra cứu và giải thích — **không** can thiệp vào quyết định nghiệp vụ cốt lõi. |

---

## 3. Mục tiêu nghiệp vụ và tiêu chí thành công

| ID | Mục tiêu nghiệp vụ | Tiêu chí thành công (đo lường được) |
|---|---|---|
| **G-01** | Loại bỏ tranh chấp về chỉ số điện nước và hóa đơn | 100% hóa đơn lưu đủ chỉ số cũ/mới, đơn giá áp dụng và thời điểm chốt; người thuê xem được toàn bộ lịch sử |
| **G-02** | Loại bỏ tranh chấp về tiền cọc | 100% hợp đồng ghi nhận số tiền cọc; mọi khấu trừ khi thanh lý đều có dòng chi tiết kèm lý do |
| **G-03** | Giảm thời gian lập hóa đơn hàng tháng của Chủ trọ | Từ ~2–3 phút/phòng (tính tay) xuống dưới 30 giây/phòng |
| **G-04** | Giảm rủi ro tin đăng giả mạo | 100% tài khoản Chủ trọ được Admin duyệt hồ sơ trước khi đăng tin |
| **G-05** | Tăng khả năng truy vết khi có khiếu nại | Mọi thao tác ảnh hưởng tới tiền (sửa giá, sửa chỉ số, xác nhận thanh toán) đều có nhật ký không thể sửa xóa |
| **G-06** | Rút ngắn thời gian tìm phòng phù hợp | Người thuê tìm được danh sách ứng viên phù hợp trong ≤ 3 thao tác hoặc 1 câu hỏi ngôn ngữ tự nhiên |

---

## 4. Phân tích Actor & Stakeholder

### 4.1 Actor trực tiếp (có tài khoản, thao tác trên hệ thống)

#### 4.1.1 Quản trị viên — Admin

**Vai trò:** Quản trị toàn bộ hệ thống và giám sát hoạt động của các bên tham gia.

**Mục tiêu:** Đảm bảo hệ thống hoạt động ổn định, thông tin đăng tải chính xác, giải quyết các vấn đề phát sinh.

**Trách nhiệm / hoạt động nghiệp vụ:**

- Quản lý tài khoản người dùng (Chủ trọ và Người thuê trọ): khóa, mở khóa, cảnh cáo.
- Xem xét hồ sơ và duyệt/từ chối yêu cầu đăng ký làm Chủ trọ.
- Kiểm duyệt tin cho thuê: ẩn tin vi phạm.
- Tiếp nhận và xử lý báo cáo, khiếu nại từ người dùng.
- Xem thống kê tổng quan hệ thống qua Dashboard (tổng người dùng, chủ trọ, người thuê, khu trọ, phòng, yêu cầu chờ duyệt).
- Tra cứu nhật ký hệ thống để đối chứng khi xử lý khiếu nại.

> **Giới hạn quyền:** Admin **không** có quyền tạo/sửa/xóa hợp đồng, hóa đơn hay xác nhận thanh toán thay Chủ trọ. Admin chỉ có quyền **đọc** các dữ liệu này, và chỉ trong phạm vi một vụ khiếu nại đang mở (xem BR-24).

#### 4.1.2 Chủ trọ — Landlord

**Vai trò:** Quản lý khu trọ, phòng trọ và trực tiếp vận hành hoạt động cho thuê của mình.

**Mục tiêu:** Tối ưu hóa việc lấp đầy phòng trống, quản lý người thuê, hợp đồng và các khoản thu chi một cách tự động, minh bạch.

**Trách nhiệm / hoạt động nghiệp vụ:**

- Gửi hồ sơ đăng ký làm Chủ trọ và chờ Admin phê duyệt.
- Quản lý thông tin cá nhân và một hoặc nhiều Khu trọ.
- Thêm, sửa, ngừng khai thác Phòng; cập nhật trạng thái khai thác và trạng thái hiển thị của Phòng.
- Đăng tin cho thuê; chủ động ẩn/hiện tin.
- Xác nhận hoặc từ chối lịch hẹn xem phòng.
- Xem và xử lý (duyệt/từ chối) yêu cầu thuê phòng.
- Ghi nhận tiền cọc và lập Hợp đồng thuê.
- Quản lý người thuê thuộc khu trọ: hợp đồng, lịch sử thanh toán, yêu cầu sửa chữa.
- Chốt chỉ số điện/nước, tạo và phát hành Hóa đơn, xác nhận tình trạng thanh toán.
- Khai báo tài khoản ngân hàng nhận tiền (không bắt buộc) để hệ thống hiển thị mã VietQR cho Người thuê chuyển khoản.
- Tiếp nhận, xử lý yêu cầu sửa chữa và gửi thông báo cho người thuê.
- Thực hiện thủ tục gia hạn, chấm dứt hợp đồng và tất toán tiền cọc.
- Xem thống kê doanh thu, phòng trống/đang thuê qua Dashboard.

#### 4.1.3 Người thuê trọ — Tenant

**Vai trò:** Tìm kiếm, đăng ký thuê phòng và quản lý quá trình thuê, sinh hoạt.

**Mục tiêu:** Tìm được phòng/người ở ghép ưng ý, theo dõi minh bạch các chi phí và hợp đồng trong quá trình thuê.

**Trách nhiệm / hoạt động nghiệp vụ:**

- Đăng ký và quản lý tài khoản cá nhân.
- Tìm kiếm, lọc phòng trọ và xem chi tiết phòng/khu trọ.
- Đặt lịch xem phòng và tham gia buổi xem.
- Gửi yêu cầu thuê phòng, theo dõi trạng thái, chủ động hủy yêu cầu.
- Nộp tiền cọc và xác nhận Hợp đồng thuê.
- Xem hóa đơn (tiền phòng, điện, nước, phí dịch vụ), chuyển khoản bằng mã VietQR của Chủ trọ nếu có, báo đã thanh toán kèm minh chứng, theo dõi lịch sử thanh toán.
- Gửi yêu cầu sửa chữa/báo sự cố và xác nhận khi sự cố đã được khắc phục.
- Nhận thông báo từ Chủ trọ và từ hệ thống.
- Tạo hồ sơ ở ghép và sử dụng tính năng tìm kiếm, kết nối với người có nhu cầu ở ghép.
- Gửi báo cáo/khiếu nại lên Admin.
- Xem thống kê cá nhân qua Dashboard.

#### 4.1.4 Trợ lý ảo — AI Assistant

**Vai trò:** Actor hệ thống phi con người, hỗ trợ người dùng tương tác với dữ liệu thông qua ngôn ngữ tự nhiên.

**Mục tiêu:** Tối ưu hóa trải nghiệm người dùng, giảm thao tác tìm kiếm thủ công, tăng tỷ lệ ghép cặp thành công.

**Trách nhiệm / hoạt động nghiệp vụ:**

- Tiếp nhận câu lệnh ngôn ngữ tự nhiên từ người dùng và phân tích ý định.
- Truy xuất dữ liệu hệ thống **trong phạm vi được phép** (xem Mục 10) để trả lời kết quả tìm kiếm phòng.
- Xuất ra văn bản giải thích lý do vì sao hai người thuê phù hợp để ở ghép, **dựa trên điểm số do hệ thống tính** (xem BR-19).
- Trả lời các truy vấn tra cứu (hợp đồng, hóa đơn của chính người hỏi) hoặc hướng dẫn sử dụng hệ thống.

> **Ranh giới quyền lực (BR-03):** AI chỉ đóng vai trò **Read-only** trong luồng nghiệp vụ lõi — tuyệt đối không có quyền Write/Update: không duyệt tài khoản, không đổi giá, không tạo/xóa hợp đồng, không xác nhận thanh toán.

### 4.2 Stakeholder gián tiếp (không có tài khoản trong Phase 1)

| Stakeholder | Liên quan như thế nào | Xử lý trong hệ thống |
|---|---|---|
| **Người ở cùng** | Ở trong phòng nhưng không đứng tên hợp đồng | Chỉ ghi nhận như thông tin đính kèm Hợp đồng (họ tên, số điện thoại), không cấp tài khoản — xem BR-11 |
| **Thợ sửa chữa** | Chủ trọ thuê ngoài để xử lý sự cố | Không tham gia hệ thống; Chủ trọ tự cập nhật trạng thái sự cố |
| **Công an khu vực / phường** | Chủ nhà trọ có nghĩa vụ pháp lý đăng ký tạm trú cho người thuê | **Out of scope** — hệ thống **không** lưu thông tin nhân thân phục vụ khai báo tạm trú. Người thuê đứng tên và Người ở cùng chỉ được ghi nhận họ tên và số điện thoại; Chủ trọ tự thu thập giấy tờ và khai báo ngoài hệ thống |
| **Ngân hàng / Ví điện tử** | Kênh chuyển tiền thực tế | Hệ thống hiển thị mã VietQR chứa tài khoản của Chủ trọ để Người thuê chuyển khoản (BR-26), nhưng **không** kết nối với ngân hàng và **không** đối soát tự động — Chủ trọ vẫn tự xác nhận đã nhận tiền |
| **Nhà cung cấp mô hình AI** | Cung cấp năng lực NLP | Phụ thuộc bên thứ ba — xem RK-02 |

---

## 5. Phân tích quy trình nghiệp vụ

### BP-01 — Đăng ký, xác thực và phê duyệt Chủ trọ

**Mục đích:** Quản lý danh tính, cấp quyền truy cập đúng vai trò; ngăn chặn tài khoản Chủ trọ giả mạo.

**Actor chính:** Admin. **Actor liên quan:** Người thuê, Chủ trọ.

**Trigger:** Người dùng mới truy cập hệ thống, hoặc người dùng hiện hữu muốn chuyển sang vai trò Chủ trọ.

**Luồng nghiệp vụ chính:**

1. Khách truy cập đăng ký tài khoản và mặc định nhận vai trò **Người thuê**.
2. Người dùng muốn làm Chủ trọ gửi **Hồ sơ đăng ký Chủ trọ**, gồm: họ tên, số CCCD kèm ảnh hai mặt, số điện thoại, và ít nhất một giấy tờ chứng minh quyền sở hữu/quản lý bất động sản (sổ đỏ, hợp đồng thuê lại, hoặc giấy phép kinh doanh nhà trọ). Người đang thuê phòng phải kết thúc hợp đồng và các yêu cầu thuê đang mở trước khi nộp (BR-01).
3. Hồ sơ chuyển sang trạng thái **Chờ duyệt**; hệ thống thông báo cho Admin.
4. Admin xem xét hồ sơ, gọi điện xác minh số điện thoại của người nộp, rồi quyết định:
   - **Duyệt** → tài khoản chuyển từ vai trò Người thuê sang vai trò Chủ trọ, số điện thoại được ghi nhận là đã xác thực; ghi nhật ký người duyệt và thời điểm duyệt.
   - **Từ chối** → bắt buộc nhập lý do; người dùng được phép bổ sung và nộp một hồ sơ mới.
5. Hệ thống gửi thông báo kết quả cho người nộp hồ sơ.
6. Người dùng đăng nhập bằng tài khoản đã được cấp quyền; hệ thống hỗ trợ đổi mật khẩu và quên mật khẩu.

**Luồng thay thế:**

- **A1 — Thu hồi quyền Chủ trọ:** Sau khi xử lý khiếu nại (BP-13), Admin có thể thu hồi vai trò Chủ trọ. Các hợp đồng đang hiệu lực **không** bị hủy tự động; tài khoản bị chặn đăng tin mới nhưng vẫn truy cập được để hoàn tất nghĩa vụ với người thuê hiện tại.
- **A2 — Khóa tài khoản Chủ trọ đang có người thuê:** Admin được cảnh báo trước khi khóa. Trong lúc bị khóa, Chủ trọ không đăng nhập được nên không xác nhận được thanh toán và nhận cọc; người thuê phải chờ tới khi được mở khóa. Đây là giới hạn được chấp nhận ở Phase 1 — khác với thu hồi vai trò ở A1, nơi Chủ trọ vẫn truy cập được để hoàn tất nghĩa vụ.

**Kết quả:** Danh tính và vai trò của mọi bên tham gia được xác lập và có thể truy vết.

---

### BP-02 — Quản lý Khu trọ và Phòng trọ

**Mục đích:** Số hóa thông tin vật lý của các bất động sản cho thuê.

**Actor chính:** Chủ trọ.

**Trigger:** Chủ trọ được duyệt và bắt đầu đưa tài sản lên hệ thống.

**Luồng nghiệp vụ chính:**

1. Chủ trọ tạo **Khu trọ**: tên, địa chỉ (tỉnh/thành và phường/xã theo danh mục đơn vị hành chính hiện hành, cùng dòng địa chỉ chi tiết), mô tả, tiện ích chung, hình ảnh.
2. Chủ trọ thêm các **Phòng** thuộc Khu trọ: mã/tên phòng, diện tích, số người tối đa, tiện ích riêng, hình ảnh, **giá thuê**, **đơn giá điện**, **đơn giá nước**, **các khoản phí dịch vụ cố định** (rác, internet, giữ xe, phí quản lý).
3. Chủ trọ chuyển **trạng thái khai thác** của phòng giữa *Trống* và *Bảo trì*; *Đang giữ chỗ* và *Đang thuê* do hệ thống đặt theo yêu cầu thuê và hợp đồng — xem Mục 8.2.
4. Chủ trọ cập nhật **trạng thái hiển thị** của phòng (Đang hiển thị / Đã ẩn) — xem Mục 8.3.
5. Khi cần ngừng khai thác vĩnh viễn, Chủ trọ **lưu trữ** phòng thay vì xóa (xem BR-09).

**Luồng thay thế:**

- **A1 — Thay đổi giá:** Chủ trọ sửa giá thuê, đơn giá điện/nước hoặc phí dịch vụ; thay đổi được ghi nhật ký (BR-23). Thay đổi **chỉ áp dụng cho hợp đồng và hóa đơn phát sinh sau đó**; hợp đồng đang hiệu lực và hóa đơn đã phát hành giữ nguyên giá đã chốt (xem BR-12, BR-13).

**Kết quả:** Toàn bộ tài sản cho thuê được số hóa với thông số giá đầy đủ.

---

### BP-03 — Đăng tin cho thuê và kiểm duyệt

**Mục đích:** Đưa phòng trống ra thị trường và duy trì chất lượng nội dung trên nền tảng.

**Actor chính:** Chủ trọ. **Actor liên quan:** Admin.

**Trigger:** Chủ trọ có phòng trống muốn cho thuê.

> **Quyết định thiết kế nghiệp vụ:** Trong Phase 1, **"Tin đăng" không phải là thực thể riêng biệt**. Một Phòng trọ ở trạng thái khai thác *Trống* và trạng thái hiển thị *Đang hiển thị* chính là một tin đăng. Cách này giảm đáng kể độ phức tạp cho nhóm 2 người và loại bỏ nguy cơ dữ liệu phòng và dữ liệu tin đăng lệch nhau. Phương án tách riêng Tin đăng (có tiêu đề, mô tả marketing, hạn hiển thị riêng) được ghi nhận là hướng mở rộng ở Phase 3.

**Luồng nghiệp vụ chính:**

1. Phòng mới tạo ở trạng thái *Đã ẩn bởi Chủ trọ*. Khi phòng sẵn sàng cho thuê, Chủ trọ bật trạng thái hiển thị cho Phòng; phòng phải có ít nhất một ảnh mới bật được.
2. Hệ thống kiểm tra điều kiện hiển thị (xem BR-05) và đưa phòng vào kết quả tìm kiếm.
3. Phòng hiển thị công khai với người tìm kiếm.
4. Chủ trọ có thể chủ động tắt hiển thị bất cứ lúc nào.

**Luồng thay thế:**

- **A1 — Admin ẩn tin vi phạm:** Sau khi xử lý báo cáo (BP-13), Admin chuyển trạng thái hiển thị sang *Đã ẩn bởi Admin*. Chủ trọ **không** tự bật lại được; phải gửi yêu cầu xem xét lại.
- **A2 — Tự động ẩn:** Khi phòng chuyển sang trạng thái khai thác *Đang giữ chỗ*, *Đang thuê* hoặc *Bảo trì*, hệ thống tự động gỡ khỏi kết quả tìm kiếm.

**Kết quả:** Thị trường phòng trống trên nền tảng luôn phản ánh đúng tình trạng thực tế.

---

### BP-04 — Tìm kiếm phòng (bộ lọc và AI)

**Mục đích:** Hỗ trợ Người thuê tìm phòng nhanh chóng và đúng nhu cầu.

**Actor chính:** Người thuê. **Actor liên quan:** AI.

**Trigger:** Người thuê có nhu cầu tìm chỗ ở.

**Luồng nghiệp vụ chính:**

1. Người thuê sử dụng bộ lọc truyền thống: khoảng giá, diện tích, khu vực, tiện ích, số người tối đa.
2. Hệ thống trả về danh sách phòng đang hiển thị, khớp điều kiện.
3. Người thuê xem chi tiết phòng và khu trọ.

**Luồng thay thế:**

- **A1 — Tìm kiếm bằng ngôn ngữ tự nhiên:** Người thuê nhập câu hỏi dạng tự nhiên (ví dụ: *"phòng dưới 3 triệu gần Đại học Công nghệ Thông tin, có gác lửng"*). AI phân tích ý định, chuyển thành điều kiện lọc, và trả về kết quả. Kết quả trả về **luôn là dữ liệu thật từ hệ thống**, AI không được tự sinh ra thông tin phòng (xem BR-18).
- **A2 — AI không hiểu ý định hoặc dịch vụ AI không khả dụng:** Hệ thống thông báo và chuyển người dùng về bộ lọc truyền thống. Tìm kiếm truyền thống phải hoạt động độc lập hoàn toàn với AI (xem BR-20).

**Kết quả:** Người thuê có danh sách phòng ứng viên.

---

### BP-05 — Đặt lịch xem phòng

**Mục đích:** Cho phép Người thuê khảo sát thực tế trước khi cam kết thuê.

**Actor chính:** Người thuê, Chủ trọ.

**Trigger:** Người thuê quan tâm một phòng cụ thể và muốn xem trực tiếp.

**Luồng nghiệp vụ chính:**

1. Người thuê chọn phòng và gửi **Yêu cầu xem phòng**, đề xuất 1–3 khung thời gian mong muốn kèm ghi chú.
2. Lịch hẹn ở trạng thái **Chờ xác nhận**; hệ thống thông báo cho Chủ trọ.
3. Chủ trọ chọn một khung thời gian và **Xác nhận**, hoặc **Đề xuất lại** khung khác, hoặc **Từ chối** kèm lý do.
4. Khi được xác nhận, hệ thống cung cấp thông tin liên hệ của hai bên cho nhau và gửi nhắc nhở trước giờ hẹn.
5. Sau buổi hẹn, một trong hai bên đánh dấu **Đã xem**.

**Luồng thay thế:**

- **A1 — Không đến hẹn:** Bên còn lại đánh dấu **Vắng mặt**. Số lần vắng mặt được ghi nhận vào hồ sơ tài khoản làm cơ sở cho Admin xử lý nếu tái diễn.
- **A2 — Hủy:** Cả hai bên đều được hủy lịch trước giờ hẹn, bắt buộc nhập lý do.

**Kết quả:** Người thuê có thông tin thực tế để quyết định; giảm tỷ lệ hủy hợp đồng ngay sau khi ký.

> **Phân kỳ:** BP-05 thuộc **Phase 3**. Nếu quỹ thời gian không cho phép, có thể lược bỏ và ghi rõ trong Đặc tả rằng việc hẹn xem phòng diễn ra ngoài hệ thống.

---

### BP-06 — Yêu cầu thuê, đặt cọc và lập Hợp đồng

**Mục đích:** Ghi nhận thỏa thuận thuê phòng chính thức, bao gồm cả khoản tiền cọc.

**Actor chính:** Người thuê, Chủ trọ.

**Trigger:** Người thuê quyết định thuê một phòng đang trống.

**Luồng nghiệp vụ chính:**

1. Người thuê gửi **Yêu cầu thuê** cho một phòng đang ở trạng thái *Trống*, kèm ngày dự kiến vào ở (không trước hôm nay) và số người dự kiến ở (không vượt số người tối đa của phòng — BR-11).
2. Yêu cầu chuyển sang trạng thái **Chờ duyệt**; hệ thống thông báo cho Chủ trọ.
3. Chủ trọ xem xét và quyết định:
   - **Từ chối** → bắt buộc nhập lý do, kết thúc luồng.
   - **Duyệt** → sang bước 4. Chỉ duyệt được khi Người thuê không đang giữ phòng khác (BR-28).
4. Khi Chủ trọ duyệt, hệ thống **đồng thời**:
   - Chuyển trạng thái khai thác của Phòng sang **Đang giữ chỗ**;
   - Gỡ phòng khỏi kết quả tìm kiếm;
   - **Tự động từ chối toàn bộ các Yêu cầu thuê khác đang chờ duyệt của cùng phòng đó**, kèm lý do "Phòng đã có người thuê khác" (xem BR-06).
5. Chủ trọ lập **Hợp đồng nháp**, trong đó **chốt cứng** tại thời điểm tạo: giá thuê/tháng, đơn giá điện, đơn giá nước, các phí dịch vụ, **số tiền cọc**, ngày bắt đầu, ngày kết thúc, hạn thanh toán, danh sách người ở cùng (nếu có). Hợp đồng nháp cũng ghi **chỉ số điện, nước lúc bàn giao phòng** — mốc tính hóa đơn đầu tiên (BR-14).
6. Người thuê xem lại Hợp đồng nháp và **Xác nhận đồng ý**, hoặc **Yêu cầu chỉnh sửa** kèm lý do — hợp đồng quay về *Nháp* để Chủ trọ sửa và gửi lại.
7. Người thuê nộp **tiền cọc** — có thể chuyển khoản bằng mã VietQR của Chủ trọ hiển thị trên hợp đồng (BR-26); Chủ trọ **xác nhận đã nhận cọc** trên hệ thống, kèm ngày nhận thực tế (không sau hôm nay) và hình thức nhận. Chủ trọ chỉ xác nhận được sau khi Người thuê đã đồng ý điều khoản; tiền đưa từ trước vẫn ghi đúng ngày nhận thực tế.
8. Khi Người thuê đã đồng ý **và** cọc đã được xác nhận, Hợp đồng chuyển sang **Đang hiệu lực**. Hợp đồng có tiền cọc bằng 0 chuyển sang **Đang hiệu lực** ngay khi Người thuê đồng ý.
9. Hệ thống chuyển trạng thái khai thác của Phòng sang **Đang thuê**.

**Luồng thay thế:**

- **A1 — Người thuê rút yêu cầu:** Trước khi Chủ trọ xử lý, hoặc sau khi được duyệt nhưng Chủ trọ chưa lập hợp đồng, Người thuê chủ động hủy → trạng thái **Đã hủy**. Nếu yêu cầu đã được duyệt, phòng trở lại **Trống** ngay và Chủ trọ nhận thông báo.
- **A2 — Yêu cầu hết hạn:** Yêu cầu thuê không được Chủ trọ xử lý trong **7 ngày** (tính đủ 168 giờ kể từ lúc gửi) tự động chuyển sang **Hết hạn**; phòng không bị giữ chỗ. Còn dưới 24 giờ tới hạn thì Chủ trọ nhận một thông báo nhắc.
- **A3 — Quá hạn giữ chỗ:** Hạn giữ chỗ là **3 ngày** (tính đủ 72 giờ) kể từ khi Chủ trọ duyệt yêu cầu thuê, gồm cả thời gian lập hợp đồng, xác nhận điều khoản và nộp cọc. Khi hạn giữ chỗ còn dưới 24 giờ, hệ thống nhắc một lần bên đang phải thao tác: Chủ trọ khi chưa lập hợp đồng hoặc hợp đồng còn *Nháp*; Người thuê khi hợp đồng chờ mình xác nhận; cả hai khi hợp đồng chờ nhận cọc — Người thuê nộp cọc, Chủ trọ xác nhận nếu đã nhận. Hết hạn mà Hợp đồng chưa *Đang hiệu lực* thì: hợp đồng (nếu đã lập) chuyển sang **Đã hủy**; yêu cầu thuê chưa được lập hợp đồng chuyển sang **Hết hạn**; phòng trở lại **Trống** và hiển thị lại.
- **A4 — Thuê ở ghép nhiều người:** Xem giới hạn tại BR-11 và Mục 13.2.
- **A5 — Chủ trọ hủy duyệt:** Sau khi duyệt nhưng chưa lập hợp đồng — chẳng hạn người thuê không đến hoặc không liên lạc được — Chủ trọ hủy duyệt, bắt buộc nhập lý do → yêu cầu chuyển sang **Từ chối**, phòng trở lại **Trống** ngay và Người thuê nhận thông báo.
- **A6 — Chủ trọ thu hồi hợp đồng đã gửi:** Hợp đồng đang chờ Người thuê xác nhận hoặc chờ nhận cọc mà Chủ trọ phát hiện sai sót — gõ nhầm giá, nhầm ngày — thì Chủ trọ thu hồi về **Nháp** để sửa và gửi lại; Người thuê nhận thông báo và phải xác nhận lại. Hạn giữ chỗ 3 ngày không đổi.

**Kết quả:** Quan hệ thuê được thiết lập chính thức với đầy đủ điều khoản tài chính đã chốt.

---

### BP-07 — Quản lý Hóa đơn và Thanh toán

**Mục đích:** Tính toán chi phí định kỳ, ghi nhận thu tiền và duy trì lịch sử đối chứng.

**Actor chính:** Chủ trọ, Người thuê.

**Trigger:** Cuối tháng. Kỳ hóa đơn là một tháng dương lịch; Chủ trọ lập được hóa đơn của một tháng từ ngày 25 của tháng đó trở đi. Chỉ lập hóa đơn định kỳ cho hợp đồng đã tới ngày bắt đầu và đang ở *Đang hiệu lực*, *Sắp hết hạn* hoặc *Đang thanh lý*.

**Luồng nghiệp vụ chính:**

1. Chủ trọ mở kỳ hóa đơn cho một phòng và **chốt chỉ số điện/nước**: hệ thống tự điền chỉ số cũ = chỉ số mới của kỳ liền trước; Chủ trọ nhập chỉ số mới kèm ảnh chụp đồng hồ (khuyến nghị).
2. Hệ thống kiểm tra tính hợp lệ (chỉ số mới ≥ chỉ số cũ — xem BR-14) và tính:
   - Tiền điện = (chỉ số điện mới − chỉ số điện cũ) × đơn giá điện đã chốt trong Hợp đồng
   - Tiền nước = (chỉ số nước mới − chỉ số nước cũ) × đơn giá nước đã chốt trong Hợp đồng
3. Hệ thống tạo **Hóa đơn tổng hợp** = tiền phòng + tiền điện + tiền nước + phí dịch vụ cố định + khoản điều chỉnh khác (nếu có, kèm mô tả).
   - Với kỳ đầu tiên hoặc kỳ cuối cùng không trọn tháng, **tiền phòng và phí dịch vụ tính theo tỷ lệ số ngày thực ở** (xem BR-15).
4. Chủ trọ **phát hành** hóa đơn → trạng thái **Chưa thanh toán**; hệ thống thông báo cho Người thuê.
5. Người thuê xem hóa đơn (thấy đủ chỉ số cũ/mới, đơn giá, cách tính), tiến hành thanh toán ngoài hệ thống — tiền mặt, hoặc chuyển khoản bằng mã VietQR của Chủ trọ hiển thị trên hóa đơn (BR-26) — rồi **báo đã thanh toán** kèm ảnh biên lai/minh chứng chuyển khoản → trạng thái **Chờ xác nhận**.
6. Chủ trọ đối soát và **Xác nhận đã thu đủ** → trạng thái **Đã thanh toán**; hoặc **Từ chối xác nhận** kèm lý do → quay lại *Chưa thanh toán*, *Thanh toán một phần* hoặc *Quá hạn* tùy số đã thu và hạn thanh toán (xem Mục 8.5).
7. Hệ thống ghi nhận vào lịch sử thanh toán và số liệu doanh thu.

**Luồng thay thế:**

- **A1 — Quá hạn:** Quá hạn thanh toán ghi trong Hợp đồng mà hóa đơn chưa được trả đủ (*Chưa thanh toán* hoặc *Thanh toán một phần*), hệ thống tự gắn cờ **Quá hạn** và gửi thông báo nhắc nhở tự động cho cả hai bên.
- **A2 — Thanh toán một phần:** Chủ trọ ghi nhận số tiền đã thu < tổng hóa đơn → trạng thái **Thanh toán một phần**, phần còn lại vẫn theo dõi công nợ.
- **A3 — Nhập sai chỉ số:** Với hóa đơn **mới nhất** của hợp đồng và **chưa** được xác nhận thanh toán, Chủ trọ được sửa; hệ thống ghi nhật ký giá trị cũ/mới, và thông báo cho Người thuê nếu hóa đơn đã phát hành. Hóa đơn cũ hơn, hoặc hóa đơn **đã** thanh toán, **không được sửa** — sai sót được điều chỉnh bằng một dòng *Điều chỉnh* ở hóa đơn kỳ kế tiếp hoặc Hóa đơn thanh lý, ghi rõ hóa đơn gốc (xem BR-16).
- **A4 — Hủy hóa đơn:** Chỉ áp dụng cho hóa đơn **mới nhất** của hợp đồng, chưa thanh toán, và bắt buộc nhập lý do.

**Kết quả:** Chi phí được tính minh bạch, có đầy đủ dữ liệu đối chứng cho mọi tranh chấp.

---

### BP-08 — Báo cáo Sự cố và Sửa chữa

**Mục đích:** Xử lý các hỏng hóc, sự cố trong quá trình lưu trú.

**Actor chính:** Người thuê, Chủ trọ. **Actor liên quan:** Admin (leo thang).

**Trigger:** Người thuê phát hiện hỏng hóc về cơ sở vật chất.

**Luồng nghiệp vụ chính:**

1. Người thuê tạo **Yêu cầu sửa chữa**: tiêu đề, mô tả, hình ảnh, mức độ khẩn cấp.
2. Yêu cầu ở trạng thái **Mới**; hệ thống thông báo cho Chủ trọ.
3. Chủ trọ tiếp nhận → trạng thái **Đang xử lý**, có thể ghi chú thời gian dự kiến hoàn thành.
4. Chủ trọ hoàn tất sửa chữa → trạng thái **Chờ người thuê xác nhận**.
5. Người thuê **Xác nhận đã khắc phục** → trạng thái **Đã đóng**.

**Luồng thay thế:**

- **A1 — Chủ trọ từ chối:** Nếu xác định lỗi do Người thuê gây ra hoặc không thuộc trách nhiệm Chủ trọ → trạng thái **Từ chối**, bắt buộc nhập lý do. Chủ trọ có thể ghi nhận **khoản bồi thường** để đưa vào hóa đơn kỳ tới hoặc khấu trừ cọc khi thanh lý.
- **A2 — Người thuê không đồng ý đã khắc phục:** Từ *Chờ người thuê xác nhận*, Người thuê chọn **Chưa đạt** kèm lý do → quay lại **Đang xử lý**.
- **A3 — Tự động đóng:** Ở trạng thái *Chờ người thuê xác nhận* quá **7 ngày** không phản hồi, hệ thống tự chuyển sang **Đã đóng**.
- **A4 — Leo thang lên Admin:** Sự cố ở trạng thái *Mới* quá **72 giờ** mà Chủ trọ không tiếp nhận, Người thuê được quyền tạo khiếu nại (BP-13) có liên kết tới sự cố này.
- **A5 — Người thuê hủy:** Người thuê tự khắc phục hoặc báo nhầm → **Đã hủy**.

**Kết quả:** Sự cố được xử lý có theo dõi, có xác nhận hai chiều, không rơi vào im lặng.

---

### BP-09 — Gia hạn Hợp đồng

**Mục đích:** Duy trì quan hệ thuê khi hợp đồng đến hạn mà hai bên vẫn muốn tiếp tục.

**Actor chính:** Chủ trọ, Người thuê.

**Trigger:** Hợp đồng chuyển sang trạng thái *Sắp hết hạn* (mặc định trước **15 ngày**).

**Luồng nghiệp vụ chính:**

1. Hệ thống tự đánh dấu Hợp đồng **Sắp hết hạn** và gửi thông báo cho cả hai bên.
2. Một trong hai bên khởi tạo **Đề nghị gia hạn**: thời hạn mới, giá thuê mới (có thể giữ nguyên hoặc thay đổi), đơn giá điện/nước mới.
3. Bên còn lại **Đồng ý** hoặc **Từ chối**.
4. Nếu đồng ý, hệ thống tạo **Hợp đồng mới** kế thừa phòng và người thuê, chốt lại toàn bộ giá tại thời điểm gia hạn; **tiền cọc được chuyển tiếp** sang hợp đồng mới (bổ sung thêm nếu giá thuê tăng).
5. Hợp đồng cũ chuyển sang **Đã kết thúc (gia hạn)**; phòng vẫn giữ trạng thái *Đang thuê*, không phải chốt số và trả phòng.
6. Nếu từ chối → một trong hai bên gửi thông báo trả phòng (BP-10). Nếu không bên nào phản hồi, hợp đồng tiếp tục hiệu lực theo điều khoản đã chốt sau ngày kết thúc cho tới khi một bên gửi thông báo trả phòng (xem Mục 8.4).

**Kết quả:** Chu kỳ thuê tiếp diễn liền mạch với điều khoản được cập nhật, lịch sử hợp đồng vẫn tách bạch.

---

### BP-10 — Chấm dứt Hợp đồng, Trả phòng và Tất toán tiền cọc

**Mục đích:** Ghi nhận việc kết thúc thỏa thuận thuê, thanh toán dứt điểm các khoản phí cuối cùng, **tất toán tiền cọc**, và hoàn trả trạng thái phòng để sẵn sàng cho chu kỳ thuê mới.

**Actor chính:** Chủ trọ, Người thuê.

**Trigger:** Hợp đồng đến hạn và không gia hạn, hoặc một trong hai bên chủ động yêu cầu chấm dứt trước thời hạn.

**Luồng nghiệp vụ chính:**

1. Người thuê hoặc Chủ trọ gửi **Thông báo trả phòng** trên hệ thống, ghi rõ ngày trả phòng dự kiến. Thời hạn báo trước tối thiểu: **30 ngày**. Thông báo gửi trước ít hơn 30 ngày vẫn được chấp nhận. Khi Người thuê là bên báo trước không đủ thời hạn, Chủ trọ được đưa phí phạt vào Hóa đơn thanh lý theo BR-22, tổng phí phạt không vượt tiền cọc. Hệ thống ghi nhận bên gửi thông báo để kiểm tra điều kiện này; Chủ trọ là bên gửi thì không có phí phạt.
2. Hợp đồng chuyển sang **Đang thanh lý**. Trong thời gian báo trước, các tháng trọn vẫn lập hóa đơn định kỳ như thường; riêng tháng chứa ngày trả phòng dự kiến không lập hóa đơn định kỳ.
3. Đến ngày trả phòng, Chủ trọ **chốt chỉ số điện/nước lần cuối** và **kiểm tra cơ sở vật chất**, ghi nhận hư hỏng kèm ảnh nếu có.
4. Chủ trọ lập **Hóa đơn thanh lý** cho khoảng từ sau kỳ hóa đơn định kỳ cuối cùng đến ngày trả phòng. Khoảng này phải nằm trong một tháng — còn thiếu tháng nào thì lập hóa đơn định kỳ tháng đó trước. Nếu tháng trả phòng đã có hóa đơn định kỳ — lập trước khi có thông báo (báo gấp), hoặc Người thuê dọn đi sớm hơn ngày dự kiến — Hóa đơn thanh lý không tính tiền phòng và phí dịch vụ, chỉ tính điện nước từ lần chốt gần nhất. Người thuê ở lại quá tháng của ngày trả phòng dự kiến thì bên gửi rút thông báo và gửi lại với ngày mới (A4). Hóa đơn định kỳ còn ở *Nháp* phải được phát hành trước. Mỗi hợp đồng chỉ có một Hóa đơn thanh lý, lập vào hoặc sau ngày trả phòng thực tế. Hóa đơn thanh lý gồm các dòng rõ ràng:

   | Khoản mục | Dấu |
   |---|---|
   | Tiền phòng kỳ cuối (tính theo số ngày thực ở) | + |
   | Tiền điện, tiền nước kỳ cuối | + |
   | Phí dịch vụ kỳ cuối | + |
   | Công nợ các kỳ trước chưa thanh toán — hệ thống tự thêm | + |
   | Phí bồi thường hư hỏng (từng khoản kèm mô tả và ảnh) | + |
   | Phí phạt chấm dứt trước hạn (nếu có, theo BR-22) | + |
   | **Tiền cọc đã nộp** | **−** |
   | **= Số dư cuối cùng** | |

   Dòng trừ tiền cọc do hệ thống tự thêm, bằng đúng số tiền cọc ghi trong Hợp đồng; Chủ trọ không tự nhập dòng này.

   Dòng công nợ cũng do hệ thống tự thêm: mỗi hóa đơn còn thiếu tiền thành một dòng bằng phần còn phải trả, và hóa đơn đó chuyển sang *Đã chuyển vào thanh lý* — không còn bị nhắc quá hạn và không thanh toán riêng được nữa. Nếu còn lượt báo thanh toán đang chờ Chủ trọ xác nhận thì chưa lập được Hóa đơn thanh lý.

5. Chủ trọ gửi bảng thanh lý cho Người thuê. Người thuê **Đồng ý**, hoặc **Chưa đồng ý** kèm lý do — bảng quay về để Chủ trọ sửa và gửi lại, không giới hạn số lần. Sau khi Người thuê đồng ý, bảng bị khóa, không sửa được nữa. Bảng đã gửi quá **7 ngày** mà Người thuê không phản hồi thì Chủ trọ được **tự chốt**, bắt buộc kèm ghi chú; kết quả như khi Người thuê đồng ý, việc tự chốt được ghi nhật ký và thông báo cho Người thuê.
6. Tất toán số dư:
   - Số dư **dương** → Người thuê thanh toán phần chênh lệch theo luồng báo đã thanh toán và Chủ trọ xác nhận như hóa đơn thường, có thể chuyển khoản bằng mã VietQR của Chủ trọ (BR-26). Hạn thanh toán tính từ lúc bảng bị khóa. Số dư dương chưa trả đủ **không** chặn việc hoàn tất thanh lý: phần còn thiếu tiếp tục được theo dõi là công nợ trên Hóa đơn thanh lý.
   - Số dư **âm** → Chủ trọ hoàn lại phần cọc còn dư và xác nhận đã hoàn, ghi rõ ngày hoàn và hình thức hoàn; số tiền hoàn do hệ thống tính từ số dư. Người thuê xem được thông tin hoàn cọc trên Hợp đồng.
7. Chủ trọ **xác nhận hoàn tất thanh lý**.
8. Hệ thống chuyển Hợp đồng sang **Đã thanh lý**.
9. Hệ thống chuyển trạng thái khai thác của Phòng sang **Bảo trì** (nếu cần dọn dẹp/sửa chữa) hoặc trực tiếp về **Trống**.

**Luồng thay thế:**

- **A1 — Không thống nhất được khoản khấu trừ:** Hợp đồng giữ nguyên trạng thái *Đang thanh lý*. Phase 1 chưa có khiếu nại nên hai bên tự giải quyết ngoài hệ thống, rồi Chủ trọ sửa bảng và gửi lại; Người thuê không phản hồi thì Chủ trọ tự chốt sau 7 ngày như bước 5. Từ Phase 2, Người thuê tạo khiếu nại (BP-13) có liên kết tới hóa đơn thanh lý, và hợp đồng giữ *Đang thanh lý* cho đến khi khiếu nại được xử lý.
- **A2 — Chấm dứt do vi phạm:** Chủ trọ chấm dứt hợp đồng do Người thuê vi phạm nghiêm trọng (không thanh toán quá 2 kỳ liên tiếp, gây hư hỏng nặng). Bắt buộc ghi lý do và bằng chứng. Trong Phase 1, Chủ trọ thực hiện bằng một thông báo trả phòng do chính mình gửi, kèm lý do; tiền cọc được khấu trừ cho công nợ và bồi thường hư hỏng theo BR-22, có thể tới toàn bộ, nhưng không có phí phạt.
- **A3 — Hợp đồng bị hủy trước khi vào ở:** Một trong hai bên hủy hợp đồng chưa tới ngày bắt đầu, bắt buộc nhập lý do → hợp đồng chuyển thẳng sang **Đã hủy**, phòng trở lại **Trống** ngay mà không cần chốt số và không lập Hóa đơn thanh lý. Nếu cọc đã nộp, tiền cọc xử lý theo BR-22: Chủ trọ hủy thì hoàn **toàn bộ**; Người thuê hủy thì Chủ trọ được giữ lại tối đa toàn bộ, ghi số tiền hoàn thực tế và lý do giữ lại. Hủy và hoàn cọc là hai bước: bên nào cũng hủy được, còn chỉ Chủ trọ ghi nhận đã hoàn cọc, kèm ngày hoàn và hình thức hoàn; việc hoàn cọc được ghi nhật ký theo BR-23. Khoản đền thêm (nếu có) khi Chủ trọ hủy do hai bên tự thỏa thuận ngoài hệ thống.
- **A4 — Rút thông báo trả phòng:** Bên đã gửi thông báo được rút khi Chủ trọ chưa lập Hóa đơn thanh lý — ví dụ Người thuê đổi ý, hoặc hai bên thỏa thuận ở tiếp. Hợp đồng quay về *Sắp hết hạn* nếu còn 15 ngày hoặc ít hơn tới ngày kết thúc (kể cả khi đã qua ngày kết thúc), ngược lại về *Đang hiệu lực*; bên còn lại nhận thông báo.

**Kết quả:** Vòng đời thuê phòng kết thúc dứt điểm về tài chính; phòng được giải phóng.

---

### BP-11 — Tìm người ở ghép

**Mục đích:** Kết nối những người có nhu cầu ở chung.

**Actor chính:** Người thuê. **Actor liên quan:** AI.

**Trigger:** Người thuê chưa có phòng muốn tìm người cùng đi thuê để chia sẻ chi phí.

**Điều kiện áp dụng:** BP-11 chỉ phục vụ Người thuê **chưa có Hợp đồng đang hiệu lực**. Hồ sơ ở ghép thể hiện **nhu cầu tìm phòng**, không gắn với một Phòng cụ thể — hai người ghép được với nhau rồi mới cùng đi tìm phòng và thực hiện BP-06. Trường hợp người đang thuê muốn tìm người vào ở cùng phòng mình **không** thuộc phạm vi hệ thống.

**Luồng nghiệp vụ chính:**

1. Người thuê tạo **Hồ sơ ở ghép**: **giới tính và yêu cầu về giới tính bạn cùng phòng**, trường học/nơi làm việc, khu vực mong muốn, khoảng ngân sách, thói quen sinh hoạt (giờ ngủ, hút thuốc, nuôi thú cưng, nấu ăn, mức độ ồn), thời điểm mong muốn chuyển vào.
2. Người thuê chọn mức độ hiển thị hồ sơ: **Công khai** hoặc **Chỉ hiện khi tôi chủ động kết nối** (xem BR-25).
3. Hệ thống tính **điểm phù hợp (%)** bằng **công thức có trọng số cố định, do hệ thống tính — không phải do AI sinh ra** (xem BR-19). Yêu cầu về giới tính là **điều kiện loại trừ cứng**, không phải điểm cộng.
4. Hệ thống hiển thị danh sách ứng viên đã sắp xếp theo điểm.
5. AI sinh **văn bản giải thích** lý do ghép cặp, dựa trên đúng các tiêu chí và trọng số mà hệ thống đã dùng.
6. Người thuê gửi **Lời mời kết nối**; ứng viên **Chấp nhận** hoặc **Từ chối**.
7. Khi cả hai chấp nhận, hệ thống hiển thị thông tin liên hệ của nhau.

**Luồng thay thế:**

- **A1 — Chặn và báo cáo:** Người dùng được chặn một hồ sơ hoặc báo cáo lên Admin (BP-13).

> **Giới hạn Phase 1 rất quan trọng:** BP-11 chỉ dừng ở **gợi ý và kết nối**. Hệ thống **không** hỗ trợ đồng thuê (nhiều người cùng đứng tên một hợp đồng) và **không** hỗ trợ chia hóa đơn giữa các bạn cùng phòng — xem BR-11 và Mục 13.2. Hai người đã kết nối thành công sẽ tự thỏa thuận: một người đứng tên hợp đồng, người còn lại được ghi nhận là *người ở cùng*.

**Kết quả:** Người thuê tìm được bạn ở ghép phù hợp tiêu chí.

---

### BP-12 — Tương tác với Trợ lý AI

**Mục đích:** Tối ưu hóa trải nghiệm sử dụng hệ thống.

**Actor chính:** Tất cả người dùng đã đăng nhập. **Actor liên quan:** AI.

**Trigger:** Người dùng cần tra cứu thông tin hoặc hướng dẫn sử dụng.

**Luồng nghiệp vụ chính:**

1. Người dùng đặt câu hỏi bằng ngôn ngữ tự nhiên (về phòng, hợp đồng, hóa đơn của chính mình, hoặc cách dùng hệ thống).
2. Hệ thống xác định vai trò và phạm vi dữ liệu mà người hỏi được phép truy cập (xem Mục 10).
3. AI truy xuất dữ liệu **trong đúng phạm vi đó** và trả lời, kèm dẫn chiếu tới bản ghi gốc (mã hóa đơn, mã hợp đồng) để người dùng tự kiểm chứng.
4. Nếu câu hỏi vượt quá phạm vi quyền hạn hoặc vượt năng lực, AI từ chối và hướng dẫn người dùng tới chức năng phù hợp.

**Luồng thay thế:**

- **A1 — Dịch vụ AI không khả dụng:** Hệ thống hiển thị thông báo và ẩn giao diện trợ lý; mọi chức năng lõi vẫn hoạt động bình thường (BR-20).

**Kết quả:** Người dùng tra cứu nhanh hơn mà không rời khỏi hệ thống.

---

### BP-13 — Báo cáo, Khiếu nại và Xử lý vi phạm

**Mục đích:** Cung cấp kênh phản ánh để Admin duy trì chất lượng nền tảng và xử lý vi phạm.

**Actor chính:** Admin. **Actor liên quan:** Người thuê, Chủ trọ.

**Trigger:** Người dùng phát hiện tin đăng giả mạo, hành vi lừa đảo, vi phạm thỏa thuận, hoặc không đồng ý với khoản khấu trừ khi thanh lý.

**Luồng nghiệp vụ chính:**

1. Người dùng tạo **Khiếu nại**: loại vi phạm, mô tả, hình ảnh, và **liên kết tới đối tượng liên quan** (phòng, hợp đồng, hóa đơn, sự cố, hoặc tài khoản).
2. Khiếu nại ở trạng thái **Chờ xử lý**.
3. Admin tiếp nhận → **Đang xem xét**. Việc tiếp nhận **mở quyền đọc có giới hạn** cho Admin đối với các bản ghi được liên kết, và bản thân việc truy cập này cũng được ghi nhật ký (xem BR-24).
4. Admin đối chiếu **nhật ký hệ thống** (BR-23) để xác định sự thật: ai đã sửa gì, lúc nào, giá trị cũ là bao nhiêu.
5. Admin ra quyết định, có thể kết hợp nhiều biện pháp:
   - Không vi phạm → **Bác bỏ** kèm giải thích.
   - Ẩn phòng/tin đăng vi phạm.
   - **Cảnh cáo** tài khoản.
   - **Khóa** tài khoản vi phạm.
   - **Thu hồi vai trò Chủ trọ**.
6. Khiếu nại chuyển sang **Đã xử lý**; hệ thống gửi thông báo kết quả cho cả người khiếu nại và người bị khiếu nại.

**Luồng thay thế:**

- **A1 — Cần bổ sung thông tin:** Admin chuyển sang **Chờ bổ sung**; quá 7 ngày người khiếu nại không phản hồi → **Đã đóng**.

> **Giới hạn:** Admin chỉ phân xử các vi phạm **quy tắc nền tảng**. Admin **không** đóng vai trò trọng tài pháp lý và không cưỡng chế được nghĩa vụ tài chính giữa hai bên.

**Kết quả:** Nền tảng được làm sạch, vi phạm được xử lý dựa trên bằng chứng.

---

## 6. Khái niệm nghiệp vụ

| Khái niệm | Mô tả nghiệp vụ |
|---|---|
| **Khu trọ** | Tập hợp các phòng trọ do một Chủ trọ quản lý, có chung địa chỉ và một số tiện ích tổng thể. |
| **Phòng trọ** | Đơn vị cho thuê cơ bản nhất, thuộc một Khu trọ, có diện tích, giá thuê, đơn giá điện/nước, phí dịch vụ, tiện ích, **trạng thái khai thác** và **trạng thái hiển thị**. Trong Phase 1, một Phòng đang hiển thị **chính là** một tin cho thuê. |
| **Hồ sơ đăng ký Chủ trọ** | Bộ giấy tờ người dùng nộp để xin cấp vai trò Chủ trọ: CCCD, số điện thoại (Admin xác minh khi duyệt), giấy tờ chứng minh quyền sở hữu/quản lý bất động sản. |
| **Lịch xem phòng** | Cuộc hẹn giữa Người thuê và Chủ trọ để khảo sát phòng thực tế trước khi quyết định thuê. |
| **Yêu cầu thuê** | Yêu cầu do Người thuê tạo để xin thuê một phòng cụ thể, cần được Chủ trọ xét duyệt. |
| **Hợp đồng** | Thỏa thuận số hóa chứng nhận quyền lưu trú của Người thuê tại Phòng trọ. **Chốt cứng** giá thuê, đơn giá điện/nước, phí dịch vụ, số tiền cọc, thời hạn và hạn thanh toán tại thời điểm tạo. |
| **Tiền cọc** | Khoản tiền Người thuê nộp trước khi vào ở, do Chủ trọ giữ để bảo đảm nghĩa vụ hợp đồng. Được khấu trừ cho các khoản còn nợ, phí bồi thường hư hỏng và phí phạt khi thanh lý; phần dư được hoàn trả. Nếu Người thuê hủy hợp đồng trước ngày vào ở, Chủ trọ được giữ lại tối đa toàn bộ cọc, kèm lý do (BR-22). Mức mặc định: **01 tháng tiền phòng**. |
| **Người ở cùng** | Người sinh sống trong phòng nhưng không đứng tên Hợp đồng. Chỉ được ghi nhận thông tin (họ tên, số điện thoại) đính kèm Hợp đồng; không có tài khoản riêng trong Phase 1. |
| **Kỳ hóa đơn** | Khoảng thời gian một hóa đơn bao phủ: một tháng dương lịch, do hệ thống xác định. Kỳ đầu tiên tính từ ngày vào ở tới cuối tháng đó; tháng có ngày trả phòng thuộc Hóa đơn thanh lý, trừ khi tháng đó đã có hóa đơn định kỳ. |
| **Chỉ số điện/nước** | Cặp giá trị (chỉ số cũ, chỉ số mới) được ghi nhận tại mỗi kỳ hóa đơn. Chỉ số mới của kỳ này là chỉ số cũ của kỳ kế tiếp; kỳ đầu tiên dùng chỉ số lúc bàn giao phòng ghi trong Hợp đồng. |
| **Hóa đơn** | Chứng từ ghi nhận khoản phải thu của một kỳ: tiền phòng + tiền điện + tiền nước + phí dịch vụ + khoản điều chỉnh. Lưu kèm chỉ số và đơn giá đã áp dụng. |
| **Hóa đơn thanh lý** | Hóa đơn đặc biệt lập khi kết thúc hợp đồng, có thêm các dòng khấu trừ tiền cọc, bồi thường hư hỏng và phí phạt; kết quả có thể là số dư dương hoặc âm. |
| **Sự cố / Yêu cầu sửa chữa** | Báo cáo từ Người thuê về vấn đề cơ sở vật chất cần Chủ trọ can thiệp xử lý. |
| **Hồ sơ ở ghép** | Tập hợp thông tin về giới tính, yêu cầu về bạn cùng phòng, thói quen sinh hoạt, nhu cầu ngân sách, trường học/công ty, phục vụ việc tính điểm phù hợp. Thuộc về một Người thuê **chưa có Hợp đồng đang hiệu lực** và không gắn với một Phòng cụ thể. |
| **Điểm phù hợp** | Giá trị phần trăm do hệ thống tính bằng công thức có trọng số cố định, thể hiện mức độ tương thích giữa hai Hồ sơ ở ghép. |
| **Thông báo** | Bản tin trong hệ thống gửi tới một người dùng cụ thể khi có sự kiện nghiệp vụ liên quan. Xem danh mục tại Mục 9. |
| **Khiếu nại** | Phản ánh của người dùng về vi phạm, gửi tới Admin, có liên kết tới đối tượng nghiệp vụ liên quan. |
| **Mã VietQR** | Mã QR theo chuẩn chuyển khoản liên ngân hàng VietQR, chứa tài khoản ngân hàng của Chủ trọ, số tiền và nội dung chuyển khoản. Người thuê quét bằng ứng dụng ngân hàng bất kỳ để chuyển tiền thẳng cho Chủ trọ. Chỉ hỗ trợ chuyển khoản, không phải bằng chứng đã thanh toán. |
| **Nhật ký hệ thống** | Bản ghi không thể sửa xóa về mọi thao tác ảnh hưởng tới tiền hoặc quyền: ai thao tác, lúc nào, giá trị trước và sau. Là cơ sở đối chứng duy nhất khi xử lý khiếu nại. |

---

## 7. Quy tắc nghiệp vụ

### 7.1 Danh tính và phân quyền

| ID | Quy tắc |
|---|---|
| **BR-01** | Người dùng muốn đóng vai trò Chủ trọ bắt buộc phải gửi hồ sơ và được Admin duyệt. Hồ sơ phải có tối thiểu: CCCD, số điện thoại, và một giấy tờ chứng minh quyền sở hữu/quản lý bất động sản. Admin xác minh số điện thoại trong lúc duyệt; duyệt xong số điện thoại được ghi nhận là đã xác thực, và người dùng đổi số điện thoại thì trạng thái này bị bỏ. Mỗi tài khoản có đúng một vai trò: khi được duyệt, tài khoản chuyển từ Người thuê sang Chủ trọ. Vì vậy tài khoản còn hợp đồng chưa kết thúc (chưa *Đã thanh lý* hoặc *Đã hủy*) hoặc còn yêu cầu thuê đang mở (*Chờ duyệt*, *Đã duyệt*) thì chưa được nộp hồ sơ; Admin cũng không duyệt được hồ sơ nếu tình trạng này phát sinh trong lúc hồ sơ chờ duyệt. |
| **BR-02** | Một Khu trọ chỉ thuộc về một Chủ trọ; một Phòng trọ chỉ thuộc về một Khu trọ. |
| **BR-03** | AI không có quyền tự ý: duyệt Chủ trọ, duyệt Người thuê, thay đổi giá, tạo/xóa hợp đồng, xác nhận thanh toán. AI là **read-only** trong mọi luồng nghiệp vụ lõi. |
| **BR-04** | Chủ trọ chỉ thao tác được trên dữ liệu thuộc Khu trọ do chính mình quản lý. Người thuê chỉ xem được hợp đồng, hóa đơn và sự cố của chính mình. |

### 7.2 Phòng và tin cho thuê

| ID | Quy tắc |
|---|---|
| **BR-05** | Một Phòng chỉ hiển thị trong kết quả tìm kiếm khi **đồng thời**: trạng thái khai thác là *Trống*, trạng thái hiển thị là *Đang hiển thị*, Khu trọ chứa phòng đang khai thác (chưa *Lưu trữ*), và Chủ trọ sở hữu đang ở trạng thái hoạt động bình thường. |
| **BR-06** | Khi một Yêu cầu thuê được duyệt, **toàn bộ Yêu cầu thuê khác đang chờ duyệt của cùng phòng phải tự động chuyển sang *Từ chối*** kèm lý do hệ thống. |
| **BR-07** | Tại một thời điểm, một Phòng chỉ có **tối đa một Hợp đồng** ở trạng thái *Đang hiệu lực*, *Sắp hết hạn* hoặc *Đang thanh lý*. |
| **BR-08** | Hệ thống chỉ cho phép chuyển trạng thái Phòng về *Trống* khi Hợp đồng hiện tại của phòng đó đã ở trạng thái *Đã thanh lý* hoặc *Đã hủy*. |
| **BR-09** | **Không được xóa vĩnh viễn** Phòng hoặc Khu trọ đã từng phát sinh Hợp đồng hoặc Hóa đơn. Chỉ được chuyển sang trạng thái *Lưu trữ*. Dữ liệu lịch sử phải được bảo toàn. |
| **BR-10** | Không được lưu trữ một Khu trọ khi còn phòng ở trạng thái *Đang giữ chỗ* hoặc *Đang thuê*. |
| **BR-27** | Một Người thuê chỉ có tối đa **một** Yêu cầu thuê ở trạng thái *Chờ duyệt* cho mỗi Phòng. |
| **BR-28** | Một Người thuê chỉ **giữ một phòng** tại một thời điểm: Chủ trọ không duyệt được yêu cầu thuê của người đang có một yêu cầu khác ở *Đã duyệt*, hoặc một hợp đồng chưa có hiệu lực (*Nháp*, *Chờ người thuê xác nhận*, *Chờ nhận cọc*). Người thuê vẫn gửi được yêu cầu cho nhiều phòng. |

### 7.3 Hợp đồng và tiền cọc

| ID | Quy tắc |
|---|---|
| **BR-11** | Mỗi Hợp đồng chỉ có **một** Người thuê đứng tên và chịu trách nhiệm toàn bộ nghĩa vụ tài chính. Những người khác ở trong phòng được ghi nhận là *Người ở cùng* — có thông tin nhưng không có nghĩa vụ trên hệ thống. Tổng số người ở không được vượt *số người tối đa* của phòng. |
| **BR-12** | Giá thuê, đơn giá điện, đơn giá nước và phí dịch vụ được **chốt cứng vào Hợp đồng** tại thời điểm tạo. Việc Chủ trọ thay đổi giá ở mức Phòng sau đó **không** ảnh hưởng tới các Hợp đồng đang hiệu lực. |
| **BR-13** | Mỗi Hóa đơn lưu lại **bản sao đơn giá đã áp dụng** tại thời điểm phát hành. Hóa đơn đã phát hành không bị tính lại khi giá thay đổi. |
| **BR-21** | Mọi Hợp đồng bắt buộc ghi nhận **số tiền cọc** (có thể bằng 0 nếu hai bên thỏa thuận không cọc). Hợp đồng chỉ chuyển sang *Đang hiệu lực* khi Người thuê đã xác nhận đồng ý điều khoản **và**, nếu tiền cọc lớn hơn 0, Chủ trọ đã xác nhận nhận đủ cọc. Thứ tự cố định: Người thuê đồng ý trước, Chủ trọ xác nhận cọc sau. |
| **BR-22** | Ngoài trường hợp Người thuê hủy hợp đồng trước ngày bắt đầu (cuối quy tắc này), tiền cọc chỉ được khấu trừ qua **Hóa đơn thanh lý**, và mọi khoản khấu trừ phải là **một dòng riêng có mô tả lý do**. Không cho phép khấu trừ một cục không giải thích. **Phí phạt** chỉ áp dụng khi Người thuê là bên gửi thông báo trả phòng, báo trước ít hơn 30 ngày và ngày trả phòng trước ngày kết thúc hợp đồng; tổng phí phạt không vượt tiền cọc. Khi Chủ trọ là bên chấm dứt thì không có phí phạt — Chủ trọ vẫn được trừ công nợ và bồi thường hư hỏng qua Hóa đơn thanh lý, phần cọc còn lại hoàn đủ cho Người thuê. Hợp đồng bị hủy trước ngày bắt đầu thì **không** lập Hóa đơn thanh lý: Chủ trọ hủy thì hoàn **toàn bộ** cọc; Người thuê hủy sau khi đã nộp cọc thì Chủ trọ được giữ lại tối đa **toàn bộ** cọc, ghi rõ số tiền hoàn thực tế và lý do giữ lại — lý do này là ghi chú bắt buộc trên hợp đồng, thay cho các dòng của Hóa đơn thanh lý. |

### 7.4 Hóa đơn và thanh toán

| ID | Quy tắc |
|---|---|
| **BR-14** | Tiền điện/nước = (chỉ số mới − chỉ số cũ) × đơn giá đã chốt. Hệ thống **từ chối** lưu khi chỉ số mới < chỉ số cũ. Chỉ số cũ của một kỳ **bắt buộc bằng** chỉ số mới của kỳ liền trước. Kỳ đầu tiên lấy chỉ số lúc bàn giao phòng ghi trong Hợp đồng: Chủ trọ nhập khi lập hợp đồng, Người thuê thấy khi xác nhận điều khoản. Chủ trọ sửa được khi hợp đồng còn ở *Nháp*, hoặc khi hợp đồng đã *Đang hiệu lực* mà chưa có hóa đơn nào — dùng khi số thực tế lúc bàn giao khác; mỗi lần sửa sau khi hợp đồng có hiệu lực đều ghi nhật ký và thông báo cho Người thuê. |
| **BR-15** | Kỳ hóa đơn đầu tiên và kỳ cuối cùng không trọn tháng thì tiền phòng **và phí dịch vụ** được tính theo **tỷ lệ số ngày thực ở** trên tổng số ngày của tháng đó, tính cả ngày vào ở và ngày trả phòng. Mọi khoản tiền làm tròn đến đồng. Ví dụ: vào ở ngày 15/10, giá thuê 3.000.000đ → 3.000.000 × 17 ÷ 31 = 1.645.161đ. |
| **BR-16** | Hóa đơn ở trạng thái *Đã thanh toán* **không được sửa đổi**. Chỉ hóa đơn mới nhất chưa thanh toán của hợp đồng mới được sửa hoặc hủy. Sai sót ở hóa đơn không còn sửa được thì điều chỉnh bằng một dòng *Điều chỉnh* có mô tả và tham chiếu tới hóa đơn gốc, đặt ở hóa đơn kỳ kế tiếp hoặc Hóa đơn thanh lý. |
| **BR-17** | Mỗi Hợp đồng chỉ có **một** Hóa đơn chưa hủy cho mỗi Kỳ hóa đơn. Hệ thống chặn việc tạo trùng kỳ. Kỳ do hệ thống xác định theo thứ tự tháng, Chủ trọ không tự chọn ngày đầu và ngày cuối kỳ. |
| **BR-06b** | Chỉ Chủ trọ mới có quyền xác nhận một Hóa đơn đã được thanh toán hoàn tất. Người thuê phải đính kèm minh chứng (ảnh biên lai/chuyển khoản) khi báo đã thanh toán, để Chủ trọ đối soát trước khi xác nhận. |
| **BR-26** | Khi Chủ trọ đã khai báo tài khoản ngân hàng nhận tiền, hệ thống hiển thị cho Người thuê đứng tên **mã VietQR** gồm tài khoản của Chủ trọ, số tiền và nội dung chuyển khoản gắn với khoản cần trả, tại ba chỗ: hợp đồng ở *Chờ nhận cọc* có tiền cọc lớn hơn 0; hóa đơn ở *Chưa thanh toán*, *Quá hạn* hoặc *Thanh toán một phần* (số tiền là phần còn phải trả); hóa đơn thanh lý có số dư dương. Mã VietQR **chỉ** hỗ trợ chuyển khoản — việc quét mã hay chuyển tiền **không** làm thay đổi trạng thái hợp đồng hay hóa đơn; khoản tiền chỉ được ghi nhận qua xác nhận nhận cọc (BR-21) hoặc xác nhận thanh toán (BR-06b). Chủ trọ chưa khai báo tài khoản thì không hiển thị mã, Người thuê thanh toán theo cách khác. |

### 7.5 Trạng thái, AI và truy vết

| ID | Quy tắc |
|---|---|
| **BR-05b** | Các trạng thái nghiệp vụ phải tuân theo **sơ đồ chuyển trạng thái được định nghĩa tại Mục 8**. Không cho phép chuyển trạng thái không có trong sơ đồ. Các chuyển trạng thái lùi hợp lệ (ví dụ *Chờ xác nhận* → *Chưa thanh toán* khi Chủ trọ từ chối) là được phép vì đã được khai báo tường minh. |
| **BR-18** | AI chỉ được trả về dữ liệu **có thật trong hệ thống** và phải dẫn chiếu tới bản ghi gốc. AI tuyệt đối không được sinh ra thông tin phòng, giá, hoặc điều khoản hợp đồng không tồn tại. |
| **BR-19** | **Điểm phù hợp ở ghép do hệ thống tính** bằng công thức có trọng số cố định, không do AI sinh. AI chỉ diễn giải điểm số đã có. Trọng số: **ngân sách 30%, thói quen sinh hoạt 30%, khu vực mong muốn 25%, trường học/nơi làm việc 15%**. Yêu cầu về giới tính bạn cùng phòng là **điều kiện lọc cứng**, không tham gia tính điểm. |
| **BR-20** | Mọi chức năng nghiệp vụ lõi (tìm kiếm bằng bộ lọc, hợp đồng, hóa đơn, sự cố) phải hoạt động **độc lập hoàn toàn** với dịch vụ AI. Khi AI không khả dụng, hệ thống chỉ mất tính năng hỗ trợ, không mất chức năng. |
| **BR-23** | Mọi thao tác thuộc các nhóm sau bắt buộc ghi **Nhật ký hệ thống** không thể sửa xóa, gồm người thực hiện, thời điểm, giá trị trước và sau: thay đổi giá thuê/đơn giá/phí dịch vụ của phòng; tạo, sửa, hủy hóa đơn; nhập/sửa chỉ số điện nước; xác nhận và từ chối xác nhận thanh toán; xác nhận nhận và hoàn cọc; tự chốt bảng thanh lý; khai báo/sửa tài khoản ngân hàng nhận tiền của Chủ trọ; duyệt/từ chối/thu hồi vai trò Chủ trọ; khóa/mở khóa tài khoản; ẩn tin đăng. |
| **BR-24** | Admin **không** có quyền đọc mặc định đối với hợp đồng và hóa đơn của người dùng. Quyền đọc chỉ được mở đối với các bản ghi **được liên kết trong một khiếu nại đang mở**, và mỗi lần truy cập đều bị ghi nhật ký. |
| **BR-25** | Hồ sơ ở ghép chỉ hiển thị **thông tin không định danh** (giới tính, khoảng ngân sách, thói quen, khu vực, trường/công ty) cho tới khi **cả hai bên chấp nhận kết nối**. Thông tin liên hệ chỉ được tiết lộ sau khi hai bên đồng ý. |

---

## 8. Vòng đời và trạng thái

### 8.1 Vòng đời Yêu cầu thuê phòng

```
                    ┌──► Từ chối (Chủ trọ không đồng ý, có lý do)
                    │
Chờ duyệt ──────────┼──► Đã duyệt ──┬──► Đã lập hợp đồng (kết thúc)
                    │               │
                    │               ├──► Từ chối (Chủ trọ hủy duyệt khi chưa lập hợp đồng, có lý do)
                    │               │
                    │               ├──► Đã hủy (Người thuê rút khi chưa lập hợp đồng)
                    │               │
                    │               └──► Hết hạn (quá 3 ngày giữ chỗ chưa lập hợp đồng)
                    │
                    ├──► Đã hủy (Người thuê rút yêu cầu)
                    │
                    └──► Hết hạn (quá 7 ngày Chủ trọ không xử lý)
```

| Trạng thái | Ý nghĩa |
|---|---|
| **Chờ duyệt** | Người thuê vừa gửi yêu cầu, Chủ trọ chưa xử lý. |
| **Đã duyệt** | Chủ trọ đồng ý cho thuê; phòng đã được giữ chỗ; đang chuẩn bị hợp đồng. |
| **Đã lập hợp đồng** | Hợp đồng đã được tạo từ yêu cầu này. Trạng thái kết thúc: hợp đồng bị hủy sau đó không làm đổi trạng thái này; Người thuê muốn thuê lại thì gửi yêu cầu mới. |
| **Từ chối** | Chủ trọ không đồng ý; Chủ trọ hủy duyệt khi chưa lập hợp đồng (BP-06 A5); hoặc bị hệ thống tự từ chối do phòng đã có người thuê khác (BR-06). |
| **Đã hủy** | Người thuê chủ động rút lại yêu cầu trước khi Chủ trọ xử lý, hoặc sau khi được duyệt nhưng chưa lập hợp đồng. |
| **Hết hạn** | Quá 7 ngày không được xử lý; hoặc đã duyệt nhưng hết hạn giữ chỗ 3 ngày mà chưa lập hợp đồng (BP-06 A3). |

### 8.2 Vòng đời Phòng trọ — Trạng thái khai thác

```
Trống ──► Đang giữ chỗ ──► Đang thuê ──► Bảo trì ──► Trống
  ▲             │                            │          
  └─────────────┘◄───────────────────────────┘          
   (hủy giữ chỗ)         (không cần bảo trì)

Trống ──► Bảo trì  (Chủ trọ tạm ngừng khai thác phòng trống để sửa chữa)

Trống / Bảo trì ──► Lưu trữ  (ngừng khai thác vĩnh viễn)
```

Chủ trọ chỉ tự chuyển phòng giữa *Trống* và *Bảo trì*, và lưu trữ phòng. Các chuyển trạng thái còn lại do hệ thống thực hiện theo yêu cầu thuê, hợp đồng và thanh lý.

| Trạng thái | Ý nghĩa |
|---|---|
| **Trống** | Sẵn sàng cho thuê, đủ điều kiện hiển thị trong tìm kiếm. |
| **Đang giữ chỗ** | Yêu cầu thuê đã được duyệt nhưng hợp đồng chưa có hiệu lực. **Không hiển thị trong tìm kiếm**, để phòng không nhận thêm yêu cầu thuê trong lúc chờ ký hợp đồng. |
| **Đang thuê** | Có hợp đồng đang hiệu lực, không khả dụng cho người tìm kiếm khác. |
| **Bảo trì** | Trạng thái đệm: đang dọn dẹp sau khi khách trả phòng, hoặc đang sửa chữa sau sự cố. Chưa sẵn sàng đón khách mới. |
| **Lưu trữ** | Phòng ngừng khai thác vĩnh viễn nhưng dữ liệu lịch sử được giữ lại (BR-09). |

### 8.3 Vòng đời Phòng trọ — Trạng thái hiển thị

> Đây là **chiều độc lập** với trạng thái khai thác: "phòng đang sửa chữa" và "phòng bị Admin ẩn tin" là hai tình huống khác nhau, phải theo dõi tách bạch.

| Trạng thái | Ai đặt | Ghi chú |
|---|---|---|
| **Đang hiển thị** | Chủ trọ | Chỉ thực sự xuất hiện trong tìm kiếm khi thỏa mãn đủ điều kiện của BR-05. |
| **Đã ẩn bởi Chủ trọ** | Chủ trọ | Chủ trọ tạm ngừng quảng bá; tự bật lại được. Phòng mới tạo mặc định ở trạng thái này. |
| **Đã ẩn bởi Admin** | Admin | Hậu quả của việc xử lý vi phạm (BP-13). Chủ trọ **không** tự bật lại được. |

### 8.4 Vòng đời Hợp đồng

```
Nháp ──► Chờ người thuê xác nhận ──► Chờ nhận cọc ──► Đang hiệu lực
 │                    │                    │                │
 └────────────────────┴────────────────────┴────────────────┼──► Đã hủy  (trước ngày bắt đầu)
                                                            │
                                         ┌──────────────────┤
                                         ▼                  ▼
                                    Sắp hết hạn ────► Đang thanh lý ──► Đã thanh lý
                                         │
                                         └──► Đã kết thúc (gia hạn) ──► [Hợp đồng mới]

Chờ người thuê xác nhận ──► Nháp  (Người thuê yêu cầu chỉnh sửa, kèm lý do)
Chờ người thuê xác nhận / Chờ nhận cọc ──► Nháp  (Chủ trọ thu hồi để sửa, BP-06 A6)
Chờ người thuê xác nhận ──► Đang hiệu lực  (tiền cọc bằng 0, Người thuê đồng ý)
Đang thanh lý ──► Đang hiệu lực / Sắp hết hạn  (bên gửi rút thông báo trả phòng, chưa lập Hóa đơn thanh lý)
```

| Trạng thái | Ý nghĩa |
|---|---|
| **Nháp** | Chủ trọ đang soạn, đang sửa theo yêu cầu chỉnh sửa của Người thuê, hoặc đã thu hồi để sửa. Người thuê xem được ở chế độ chỉ đọc nhưng chưa xác nhận được; vẫn hủy được nếu đổi ý. |
| **Chờ người thuê xác nhận** | Người thuê đang xem lại điều khoản. |
| **Chờ nhận cọc** | Người thuê đã đồng ý; đang chờ nộp và xác nhận tiền cọc, trong hạn giữ chỗ 3 ngày tính từ khi Chủ trọ duyệt yêu cầu thuê (BP-06 A3). |
| **Đang hiệu lực** | Đã đủ điều kiện BR-21. Từ ngày bắt đầu trở đi là giai đoạn lưu trú; trước ngày bắt đầu, hợp đồng đã ràng buộc hai bên và phòng đã bị chiếm dụng. |
| **Sắp hết hạn** | Hệ thống tự đánh dấu hợp đồng *Đang hiệu lực* trước 15 ngày, kích hoạt thông báo gia hạn (BP-09); hợp đồng đang thanh lý không chuyển sang trạng thái này. Qua ngày kết thúc mà chưa bên nào gửi thông báo trả phòng thì hợp đồng giữ trạng thái này và tiếp tục hiệu lực theo điều khoản đã chốt, hóa đơn định kỳ vẫn được lập như thường. |
| **Đang thanh lý** | Đã có thông báo trả phòng; đang chốt số cuối và tất toán cọc. |
| **Đã thanh lý** | Hoàn tất thủ tục trả phòng, đã chốt phí cuối và xử lý xong tiền cọc. |
| **Đã kết thúc (gia hạn)** | Kết thúc do được thay thế bởi hợp đồng gia hạn; không phải trả phòng. |
| **Đã hủy** | Hủy trước ngày bắt đầu hợp đồng: người thuê không đồng ý điều khoản, hết hạn giữ chỗ mà hợp đồng chưa đủ điều kiện hiệu lực, hoặc một trong hai bên hủy sau khi hợp đồng đã hiệu lực nhưng chưa tới ngày vào ở. Tiền cọc đã nhận xử lý theo BR-22: Chủ trọ hủy thì hoàn **toàn bộ**; Người thuê hủy thì Chủ trọ được giữ lại tối đa toàn bộ. |

### 8.5 Vòng đời Hóa đơn

```
                      (người thuê báo đã trả)
                ┌──────────────────────────► Chờ xác nhận ──► Đã thanh toán
                │                                  │
                │      (xác nhận còn thiếu /       │
                │       từ chối xác nhận)          ▼
Nháp ──► [ Chưa thanh toán · Thanh toán một phần · Quá hạn ]
                │                       │
                │                       └──► Đã chuyển vào thanh lý  (khi lập Hóa đơn thanh lý)
                │
                └──► Đã hủy  (chỉ từ Chưa thanh toán)
```

Các chuyển trạng thái hợp lệ:

| Từ | Sang | Khi nào |
|---|---|---|
| Nháp | Chưa thanh toán | Chủ trọ phát hành |
| Chưa thanh toán | Đã hủy | Chủ trọ hủy, bắt buộc có lý do |
| Chưa thanh toán, Thanh toán một phần, Quá hạn | Chờ xác nhận | Người thuê báo đã thanh toán kèm minh chứng |
| Chờ xác nhận | Đã thanh toán | Chủ trọ xác nhận và tổng đã thu bằng tổng hóa đơn |
| Chờ xác nhận | Thanh toán một phần hoặc Quá hạn | Chủ trọ xác nhận nhưng tổng đã thu còn thiếu: *Quá hạn* nếu đã qua hạn thanh toán, ngược lại *Thanh toán một phần* |
| Chờ xác nhận | Chưa thanh toán, Thanh toán một phần hoặc Quá hạn | Chủ trọ từ chối xác nhận: *Quá hạn* nếu đã qua hạn thanh toán; ngược lại *Thanh toán một phần* nếu đã thu được một phần; còn lại *Chưa thanh toán* |
| Chưa thanh toán, Thanh toán một phần | Quá hạn | Qua hạn thanh toán mà chưa trả đủ — hệ thống tự gắn cờ |
| Chưa thanh toán, Thanh toán một phần, Quá hạn | Đã chuyển vào thanh lý | Chủ trọ lập Hóa đơn thanh lý; phần còn nợ thành một dòng công nợ của Hóa đơn thanh lý |

| Trạng thái | Ý nghĩa |
|---|---|
| **Nháp** | Chủ trọ đã chốt số nhưng chưa phát hành; còn sửa được tự do. |
| **Chưa thanh toán** | Đã phát hành, chờ Người thuê đóng tiền. |
| **Chờ xác nhận** | Người thuê đã báo đã thanh toán kèm minh chứng; chờ Chủ trọ đối soát. Đây là nơi BR-06b được thực thi: hóa đơn không tự sang *Đã thanh toán* khi chưa có xác nhận của Chủ trọ. |
| **Thanh toán một phần** | Chủ trọ đã xác nhận thu được một phần, chưa qua hạn thanh toán; phần còn lại theo dõi công nợ. |
| **Quá hạn** | Qua hạn thanh toán ghi trong hợp đồng mà chưa trả đủ, kể cả khi đã trả một phần — số đã trả vẫn được ghi nhận; hệ thống tự gắn cờ và gửi nhắc nhở. |
| **Đã thanh toán** | Chủ trọ đã xác nhận nhận đủ tiền. Không được sửa (BR-16). |
| **Đã hủy** | Hóa đơn lập sai, hủy trước khi thanh toán, có ghi lý do. |
| **Đã chuyển vào thanh lý** | Phần còn nợ đã được đưa vào Hóa đơn thanh lý để trừ vào tiền cọc; không còn là công nợ riêng. |
| **Chờ người thuê xác nhận** | Chỉ với Hóa đơn thanh lý: Chủ trọ đã gửi bảng thanh lý, chờ Người thuê đồng ý. |
| **Chờ hoàn cọc** | Chỉ với Hóa đơn thanh lý có số dư âm: Người thuê đã đồng ý, chờ Chủ trọ hoàn phần cọc dư. |

**Hóa đơn thanh lý** đi qua thêm bước gửi và đồng ý trước khi vào luồng thanh toán:

```
Nháp ──► Chờ người thuê xác nhận ──┬──► Chưa thanh toán ──► … ──► Đã thanh toán   (số dư dương)
  ▲                │                ├──► Chờ hoàn cọc ──► Đã thanh toán            (số dư âm)
  └────────────────┘                └──► Đã thanh toán                             (số dư bằng 0)
   (Người thuê chưa đồng ý, kèm lý do)
```

Quá 7 ngày kể từ lần gửi gần nhất mà Người thuê không phản hồi, Chủ trọ được tự chốt kèm ghi chú; bảng bị khóa và đi tiếp như khi Người thuê đồng ý. Hợp đồng được hoàn tất thanh lý khi bảng đã khóa, kể cả khi số dư dương chưa trả đủ — Hóa đơn thanh lý khi đó vẫn mở như một khoản nợ.

Với Hóa đơn thanh lý, *Đã thanh toán* nghĩa là đã tất toán xong: Người thuê trả đủ, Chủ trọ hoàn đủ, hoặc số dư bằng 0.

### 8.6 Vòng đời Sự cố / Yêu cầu sửa chữa

```
Mới ──► Đang xử lý ──► Chờ người thuê xác nhận ──► Đã đóng
 │            ▲                   │
 │            └───────────────────┘ (người thuê báo "Chưa đạt")
 │
 ├──► Từ chối (không thuộc trách nhiệm Chủ trọ, có lý do)
 └──► Đã hủy (người thuê tự khắc phục / báo nhầm)
```

### 8.7 Vòng đời Lịch xem phòng

```
Chờ xác nhận ──► Đã xác nhận ──► Đã xem
      │                │
      │                ├──► Vắng mặt
      │                └──► Đã hủy
      ├──► Đề xuất lại ──► Chờ xác nhận
      └──► Từ chối
```

### 8.8 Vòng đời Hồ sơ đăng ký Chủ trọ

```
Chờ duyệt ──► Đã duyệt ──► Đã thu hồi
     │
     └──► Từ chối  (kết thúc — người dùng bổ sung và nộp một hồ sơ mới, bắt đầu lại từ Chờ duyệt)
```

### 8.9 Vòng đời Khiếu nại

```
Chờ xử lý ──► Đang xem xét ──► Đã xử lý
                   │
                   ├──► Bác bỏ
                   └──► Chờ bổ sung ──► Đang xem xét
                              │
                              └──► Đã đóng (quá 7 ngày không phản hồi)
```

---

## 9. Danh mục sự kiện thông báo

> Bảng dưới đây xác định mỗi sự kiện nghiệp vụ gửi thông báo cho ai và ở mức độ nào.

| Sự kiện nghiệp vụ | Người nhận | Mức độ |
|---|---|---|
| Hồ sơ Chủ trọ mới được nộp | Admin | Thường |
| Hồ sơ Chủ trọ được duyệt / bị từ chối | Người nộp hồ sơ | Cao |
| Có yêu cầu xem phòng mới | Chủ trọ | Cao |
| Lịch xem phòng được xác nhận / đề xuất lại / từ chối | Người thuê | Cao |
| Nhắc nhở trước giờ hẹn xem phòng | Cả hai bên | Thường |
| Có yêu cầu thuê mới | Chủ trọ | Cao |
| Yêu cầu thuê được duyệt / bị từ chối | Người thuê | Cao |
| Người thuê rút yêu cầu thuê đã được duyệt | Chủ trọ | Cao |
| Yêu cầu thuê hết hạn | Người thuê (thêm Chủ trọ khi hết hạn giữ chỗ) | Cao |
| Yêu cầu thuê sắp hết hạn xử lý (còn 1 ngày) | Chủ trọ | Thường |
| Hợp đồng nháp được gửi để xác nhận | Người thuê | Cao |
| Người thuê yêu cầu chỉnh sửa hợp đồng (kèm lý do) | Chủ trọ | Cao |
| Chủ trọ thu hồi hợp đồng đã gửi để sửa | Người thuê | Cao |
| Nhắc xác nhận hợp đồng và nộp tiền cọc (hạn giữ chỗ còn 1 ngày) | Người thuê | Cao |
| Nhắc lập, gửi hợp đồng hoặc xác nhận cọc (hạn giữ chỗ còn 1 ngày) | Chủ trọ | Cao |
| Tiền cọc được xác nhận, hợp đồng có hiệu lực | Cả hai bên | Cao |
| Hợp đồng bị hủy | Bên còn lại (cả hai bên khi hệ thống hủy do hết hạn giữ chỗ) | Cao |
| Chỉ số đầu của hợp đồng được sửa | Người thuê | Cao |
| Chủ trọ ghi nhận đã hoàn cọc | Người thuê | Cao |
| Hóa đơn mới được phát hành | Người thuê | Cao |
| Hóa đơn sắp đến hạn (trước 3 ngày) | Người thuê | Thường |
| Hóa đơn quá hạn | Cả hai bên | Cao |
| Người thuê báo đã thanh toán | Chủ trọ | Cao |
| Thanh toán được xác nhận / bị từ chối xác nhận | Người thuê | Cao |
| Hóa đơn được điều chỉnh | Người thuê | Cao |
| Hóa đơn bị hủy | Người thuê | Cao |
| Sự cố mới được báo | Chủ trọ | Cao |
| Sự cố đổi trạng thái | Người thuê | Thường |
| Sự cố quá 72 giờ chưa tiếp nhận | Người thuê (gợi ý leo thang) | Cao |
| Hợp đồng sắp hết hạn (trước 15 ngày) | Cả hai bên | Cao |
| Đề nghị gia hạn được gửi / được phản hồi | Bên còn lại | Cao |
| Thông báo trả phòng được gửi | Bên còn lại | Cao |
| Thông báo trả phòng bị rút | Bên còn lại | Cao |
| Bảng thanh lý được gửi để xác nhận | Người thuê | Cao |
| Người thuê chưa đồng ý bảng thanh lý (kèm lý do) | Chủ trọ | Cao |
| Chủ trọ tự chốt bảng thanh lý | Người thuê | Cao |
| Hoàn tất thanh lý và tất toán cọc | Cả hai bên | Cao |
| Lời mời kết nối ở ghép / được chấp nhận | Người nhận lời mời / người gửi | Thường |
| Khiếu nại được tiếp nhận / có kết quả xử lý | Người khiếu nại và người bị khiếu nại | Cao |
| Tài khoản bị cảnh cáo / khóa / thu hồi vai trò | Người bị xử lý | Cao |
| Tài khoản được mở khóa | Người được mở khóa | Cao |

> **Kênh gửi:** Phase 1 chỉ dùng thông báo trong hệ thống (in-app). Email và Zalo/SMS nằm ngoài phạm vi.

---

## 10. Ranh giới quyền hạn của Trợ lý AI

> BR-03 giới hạn quyền **ghi** của AI. Mục này giới hạn quyền **đọc** — yếu tố quyết định rủi ro rò rỉ dữ liệu.

### 10.1 Phạm vi dữ liệu AI được truy xuất, theo vai trò người hỏi

| Loại dữ liệu | Người thuê hỏi | Chủ trọ hỏi | Admin hỏi |
|---|---|---|---|
| Phòng đang hiển thị công khai | ✅ Toàn bộ | ✅ Toàn bộ | ✅ Toàn bộ |
| Phòng thuộc khu trọ của mình | — | ✅ | ❌ |
| Hợp đồng của chính mình | ✅ | ✅ (của khu trọ mình) | ❌ |
| Hóa đơn của chính mình | ✅ | ✅ (của khu trọ mình) | ❌ |
| Sự cố của chính mình | ✅ | ✅ (của khu trọ mình) | ❌ |
| Hợp đồng/hóa đơn của người khác | ❌ | ❌ | ❌ |
| Hồ sơ ở ghép của người khác | ⚠️ Chỉ phần không định danh (BR-25) | ❌ | ❌ |
| Thông tin liên hệ người dùng khác | ❌ (trừ khi đã kết nối, yêu cầu thuê đã được duyệt hoặc đã có hợp đồng) | ⚠️ Chỉ người thuê của mình | ❌ |
| Nhật ký hệ thống | ❌ | ❌ | ⚠️ Chỉ trong khiếu nại đang mở (BR-24) |
| Tài liệu hướng dẫn sử dụng | ✅ | ✅ | ✅ |

### 10.2 Nguyên tắc vận hành

| Nguyên tắc | Nội dung |
|---|---|
| **Lọc trước, sinh sau** | Hệ thống phải lọc dữ liệu theo quyền của người hỏi **trước khi** đưa vào AI. Không đưa dữ liệu ngoài phạm vi rồi trông chờ AI tự kiềm chế. |
| **Luôn dẫn chiếu nguồn** | Mọi câu trả lời về hợp đồng/hóa đơn phải kèm mã bản ghi để người dùng tự mở và kiểm chứng. |
| **Không tự sinh dữ liệu** | AI không được bịa thông tin phòng, giá hay điều khoản (BR-18). |
| **Trách nhiệm** | Câu trả lời của AI mang tính **tham khảo**. Bản ghi gốc trong hệ thống là căn cứ duy nhất khi có tranh chấp. Giao diện phải thể hiện rõ điều này. |
| **Suy biến an toàn** | Khi dịch vụ AI lỗi hoặc hết hạn mức, hệ thống ẩn trợ lý và giữ nguyên toàn bộ chức năng lõi (BR-20). |

---

## 11. Giả định và Ràng buộc

### 11.1 Giả định

| ID | Giả định | Rủi ro nếu sai |
|---|---|---|
| **AS-01** | Mỗi phòng có đồng hồ điện và đồng hồ nước riêng, đọc được chỉ số. | Nếu dùng đồng hồ tổng chia đều theo đầu người, toàn bộ công thức BR-14 phải thay đổi. |
| **AS-02** | Chủ trọ và Người thuê tự thực hiện việc chuyển tiền ngoài hệ thống. | Nếu bắt buộc thanh toán trong hệ thống, phải mở rộng phạm vi sang tích hợp cổng thanh toán. |
| **AS-03** | Hợp đồng trên hệ thống mang tính **ghi nhận thỏa thuận nội bộ**, không thay thế hợp đồng giấy có giá trị pháp lý. | Nếu cần giá trị pháp lý, phải bổ sung chữ ký số — vượt xa năng lực nhóm 2 người. |
| **AS-04** | Một chủ trọ quản lý dưới 50 phòng; hệ thống phục vụ quy mô đồ án, không phải quy mô thương mại. | Ảnh hưởng tới thiết kế hiệu năng ở giai đoạn sau. |
| **AS-05** | Người dùng đều truy cập qua trình duyệt trên máy tính hoặc điện thoại; không cần ứng dụng cài đặt. | — |
| **AS-06** | Người thuê và Chủ trọ đã gặp mặt trực tiếp trước khi ký hợp đồng (dù bước hẹn xem phòng có nằm trong hệ thống hay không). | — |

### 11.2 Ràng buộc

| ID | Ràng buộc |
|---|---|
| **CO-01** | Nhân lực: **02 thành viên**. |
| **CO-02** | Thời gian: tối đa **2.5 tháng**. |
| **CO-03** | Nền tảng: **Web**, không phát triển ứng dụng di động native. |
| **CO-04** | Công nghệ định hướng theo yêu cầu của Đồ án 1 (xem Mục 15). |
| **CO-05** | Kiến trúc: giao diện Web **không truy cập cơ sở dữ liệu trực tiếp**, giao tiếp hoàn toàn qua Backend API. |
| **CO-06** | Không tự ý thêm chức năng nằm ngoài nghiệp vụ. Mọi chức năng mới phải có cơ sở từ nghiệp vụ, yêu cầu hệ thống hoặc tiêu chí môn học. |

### 11.3 Rủi ro

| ID | Rủi ro | Mức độ | Biện pháp giảm thiểu |
|---|---|---|---|
| **RK-01** | Phạm vi quá lớn so với 2 người / 2.5 tháng → không hoàn thiện được luồng cốt lõi. | **Cao** | Phân kỳ nghiêm ngặt theo Mục 13.3; Phase 1 phải chạy được end-to-end trước khi động vào Phase 2. |
| **RK-02** | Phụ thuộc dịch vụ AI bên thứ ba (chi phí, hạn mức, độ trễ, chất lượng trả lời). | **Cao** | BR-20 bảo đảm hệ thống vẫn dùng được khi không có AI; giới hạn số lượt gọi mỗi người dùng. |
| **RK-03** | Nghiệp vụ tiền bạc phức tạp (cọc, tháng lẻ, công nợ) dễ sinh lỗi tính toán. | **Cao** | Viết kiểm thử riêng cho khối tính hóa đơn và thanh lý; các quy tắc BR-12 → BR-17 phải được kiểm thử tự động. |
| **RK-04** | Tính năng ở ghép dễ bị phình to nếu hỗ trợ đồng thuê. | Trung bình | Chốt giới hạn tại BR-11; ghi rõ vào Out of scope. |
| **RK-05** | Dữ liệu nhân thân (CCCD, ảnh giấy tờ) là dữ liệu nhạy cảm. | Trung bình | Hạn chế quyền đọc theo BR-04 và BR-24; chỉ lưu những gì thực sự cần cho việc duyệt hồ sơ. |

---

## 12. Nền tảng

Dự án triển khai trên **một nền tảng duy nhất: ứng dụng Web**. Cả ba actor người dùng — Admin, Chủ trọ, Người thuê — đăng nhập và thao tác qua trình duyệt, với giao diện phân hóa theo vai trò (role-based).

**Hệ quả nghiệp vụ:**

- Giao diện phải sử dụng được tốt trên **màn hình điện thoại**, vì Người thuê (chủ yếu là sinh viên và người lao động trẻ) tìm phòng và xem hóa đơn chủ yếu trên điện thoại. Đây là ràng buộc nghiệp vụ, không phải lựa chọn kỹ thuật.
- Nghiệp vụ nhập liệu nặng của Chủ trọ (chốt số điện nước hàng loạt, lập hóa đơn) được tối ưu cho màn hình lớn.
- Người thuê cần chụp và tải ảnh trực tiếp từ điện thoại trong hai luồng: báo sự cố (BP-08) và gửi minh chứng thanh toán (BP-07).

---

## 13. Ranh giới hệ thống

### 13.1 Trong phạm vi — In Scope

- Đăng ký, đăng nhập, phân quyền theo vai trò.
- Admin duyệt hồ sơ Chủ trọ, quản lý người dùng, xem thống kê.
- Chủ trọ quản lý Khu trọ, Phòng trọ, trạng thái khai thác và hiển thị.
- Người thuê tìm kiếm phòng bằng bộ lọc và bằng ngôn ngữ tự nhiên.
- Đặt lịch xem phòng.
- Yêu cầu thuê, ghi nhận tiền cọc, lập và quản lý Hợp đồng.
- Gia hạn hợp đồng.
- Chốt chỉ số điện/nước, lập và phát hành Hóa đơn, **ghi nhận và xác nhận thanh toán**.
- Hiển thị mã VietQR chứa tài khoản của Chủ trọ để Người thuê chuyển khoản tiền cọc, tiền hóa đơn và số dư thanh lý (BR-26).
- Chấm dứt hợp đồng, lập Hóa đơn thanh lý và tất toán tiền cọc.
- Báo cáo sự cố và theo dõi sửa chữa có xác nhận hai chiều.
- Tạo hồ sơ và gợi ý kết nối người ở ghép, có điểm phù hợp và giải thích bằng AI.
- Trợ lý AI tra cứu, tìm kiếm NLP, giải thích và hướng dẫn, trong phạm vi quyền hạn tại Mục 10.
- Hệ thống thông báo trong ứng dụng theo danh mục tại Mục 9.
- Nhật ký hệ thống cho các thao tác nhạy cảm theo BR-23.
- Dashboard thống kê cho cả ba vai trò.
- Báo cáo, khiếu nại và xử lý vi phạm.

### 13.2 Ngoài phạm vi — Out of Scope

> Đây là các quyết định **dứt khoát**, không kèm điều kiện. Một hạng mục đã nằm ngoài phạm vi chỉ được đưa trở lại bằng một quyết định mới, không tự động quay lại vì "có vẻ cần".

| # | Nội dung loại trừ | Lý do |
|---|---|---|
| 1 | **Tích hợp cổng thanh toán trực tuyến và đối soát tự động với ngân hàng.** Hệ thống chỉ hiển thị mã VietQR của Chủ trọ để hỗ trợ chuyển khoản (BR-26), và ghi nhận thanh toán qua cơ chế "Người thuê báo đã trả kèm minh chứng → Chủ trọ xác nhận". | Vượt quá năng lực và thời gian; cần tài khoản doanh nghiệp thật. |
| 2 | **Đồng thuê (nhiều người cùng đứng tên một hợp đồng) và chia hóa đơn giữa các bạn cùng phòng.** Mỗi hợp đồng chỉ có một người đứng tên (BR-11). | Làm phình mô hình dữ liệu và toàn bộ luồng thanh toán. Đây là giới hạn có ý thức, không phải thiếu sót. |
| 3 | **Chữ ký số và giá trị pháp lý của hợp đồng.** Hợp đồng trên hệ thống chỉ ghi nhận thỏa thuận (AS-03). | Yêu cầu hạ tầng pháp lý và chứng thư số. |
| 4 | **Khai báo tạm trú tạm vắng với cơ quan công an.** Hệ thống không lưu thông tin nhân thân phục vụ khai báo; Chủ trọ tự thu thập giấy tờ và khai báo ngoài hệ thống (xem Mục 4.2). | Cần tích hợp với hệ thống của cơ quan nhà nước. |
| 5 | **AI tự động ra quyết định nghiệp vụ** (tự duyệt hồ sơ, tự thu tiền, tự chấm dứt hợp đồng). | Vi phạm BR-03; rủi ro nghiệp vụ không chấp nhận được. |
| 6 | **Ứng dụng di động native (Android/iOS).** Nền tảng chốt là Web (CO-03). | Ràng buộc đề bài. |
| 7 | **Nhắn tin trực tiếp (chat) giữa Chủ trọ và Người thuê.** Hai bên trao đổi qua thông tin liên hệ được tiết lộ sau khi Chủ trọ duyệt yêu cầu thuê, sau khi xác nhận lịch xem phòng, hoặc sau khi kết nối ở ghép. | Chat thời gian thực là một hệ thống con riêng, vượt quỹ thời gian. |
| 8 | **Đánh giá, xếp hạng chủ trọ/người thuê (review & rating).** | Cần khối lượng người dùng thật mới có ý nghĩa; dễ bị lạm dụng. |
| 9 | **Tách "Tin đăng" thành thực thể riêng** có tiêu đề, nội dung marketing và hạn hiển thị độc lập với Phòng. | Quyết định tại BP-03; hướng mở rộng Phase 3. |
| 10 | **Quản lý tài sản/nội thất chi tiết trong phòng** (kiểm kê từng món khi giao/nhận phòng). | Hư hỏng được xử lý ở mức mô tả tự do trong Hóa đơn thanh lý. |

### 13.3 Phân kỳ triển khai

**Nhận định rủi ro:** Với 02 thành viên và 2.5 tháng, phạm vi In-scope vẫn là rất lớn. Nếu dàn trải đều, dự án có nguy cơ không hoàn thiện được luồng cốt lõi. Giai đoạn *Requirements Specification* tiếp theo bắt buộc phải bám sát phân kỳ dưới đây.

#### Phase 1 — Must-have / Core (~60% quỹ thời gian)

Mục tiêu: **một vòng đời thuê phòng chạy được trọn vẹn từ đầu đến cuối.**

| BP | Nội dung |
|---|---|
| BP-01 | Đăng ký, xác thực, duyệt Chủ trọ |
| BP-02 | Quản lý Khu trọ và Phòng trọ (đầy đủ giá và đơn giá) |
| BP-03 | Đăng/ẩn tin cho thuê (Chủ trọ tự bật/tắt; Admin xử lý vi phạm bằng khóa tài khoản Chủ trọ) |
| BP-04 | Tìm kiếm bằng **bộ lọc truyền thống** (chưa có AI) |
| BP-06 | Yêu cầu thuê, đặt cọc, lập Hợp đồng |
| BP-07 | Chốt số, lập Hóa đơn, xác nhận thanh toán, thanh toán một phần và theo dõi công nợ |
| BP-10 | Chấm dứt hợp đồng, thanh lý, tất toán cọc |
| — | Thông báo trong ứng dụng cho các sự kiện mức **Cao** |
| — | Nhật ký hệ thống theo BR-23 |
| — | Dashboard cơ bản cho 3 vai trò |

> Phase 1 **phải** bao gồm tiền cọc và thanh lý. Một hệ thống quản lý trọ không xử lý được tiền cọc thì chưa giải quyết được nỗi đau chính của người dùng.

#### Phase 2 — Value-added (~30% quỹ thời gian)

| BP | Nội dung |
|---|---|
| BP-03 A1 | Admin ẩn tin vi phạm, làm cùng BP-13 |
| BP-04 A1 | Tìm kiếm bằng ngôn ngữ tự nhiên (AI) |
| BP-08 | Báo cáo sự cố và sửa chữa |
| BP-11 | Tìm người ở ghép + điểm phù hợp + giải thích bằng AI |
| BP-12 | Trợ lý AI tra cứu |
| BP-13 | Báo cáo, khiếu nại và xử lý vi phạm |
| — | Dashboard thống kê nâng cao |

#### Phase 3 — Nếu còn thời gian (~10%)

| BP | Nội dung |
|---|---|
| BP-05 | Đặt lịch xem phòng |
| BP-09 | Gia hạn hợp đồng |
| — | Tách Tin đăng thành thực thể riêng |

> **Nguyên tắc dừng:** Không bắt đầu Phase 2 khi Phase 1 chưa chạy được end-to-end trên dữ liệu thật. Nếu buộc phải cắt, cắt từ Phase 3 lên.

---

## 14. Yêu cầu chất lượng ở mức nghiệp vụ

> Đây **không** phải đặc tả kỹ thuật, mà là các kỳ vọng nghiệp vụ mà thiết kế ở giai đoạn sau phải đáp ứng.

| ID | Yêu cầu | Lý do nghiệp vụ |
|---|---|---|
| **QR-01** | Mọi con số tiền hiển thị cho Người thuê phải kèm **cách tính** (chỉ số cũ, chỉ số mới, đơn giá). | Minh bạch là giá trị cốt lõi của hệ thống (G-01). |
| **QR-02** | Dữ liệu tài chính (hợp đồng, hóa đơn, thanh toán, cọc) **không được xóa cứng** trong bất kỳ hoàn cảnh nào. | Cơ sở đối chứng khi tranh chấp (G-05). |
| **QR-03** | Nhật ký hệ thống phải **không sửa xóa được**, kể cả bởi Admin. | Nếu Admin sửa được nhật ký thì nhật ký mất giá trị đối chứng. |
| **QR-04** | Ảnh giấy tờ nhân thân chỉ hiển thị cho Admin trong quá trình duyệt hồ sơ, không hiển thị công khai ở bất kỳ đâu. | Bảo vệ dữ liệu nhạy cảm (RK-05). |
| **QR-05** | Giao diện phải dùng được trên màn hình rộng từ 360px. | Người thuê chủ yếu dùng điện thoại (Mục 12). |
| **QR-06** | Mọi thao tác không thể hoàn tác (hủy hợp đồng, xác nhận thanh lý, khóa tài khoản) phải có bước xác nhận rõ ràng. | Giảm sai sót thao tác trong nghiệp vụ tiền bạc. |
| **QR-07** | Thông tin liên hệ cá nhân chỉ được tiết lộ khi có cơ sở nghiệp vụ (yêu cầu thuê đã được Chủ trọ duyệt, hợp đồng chưa kết thúc, lịch hẹn đã xác nhận, kết nối ở ghép đã đồng thuận). | Chống thu thập dữ liệu và quấy rối (BR-25). |

---

## 15. Bối cảnh môn học

Thông tin dưới đây được giữ nguyên theo tài liệu mô tả dự án gốc.

| Hạng mục | Nội dung |
|---|---|
| **Môn phụ trách** | Đồ án 1 |
| **Thành viên** | Giang, Vinh (tổng 02 thành viên) |
| **Thời gian thực hiện** | Tối đa 2.5 tháng |
| **Nền tảng** | Web |

**Công nghệ định hướng:**

| Lớp | Công nghệ |
|---|---|
| **Backend** | ASP.NET Core Web API, Entity Framework Core (ORM), ASP.NET Identity, FluentValidation, Serilog, Swagger/OpenAPI |
| **Database** | Supabase (PostgreSQL hosting) |
| **Authentication** | JWT Authentication |
| **Trợ lý AI** | Google Gemini API |
| **Frontend** | React + TypeScript, Tailwind CSS, React Hook Form, TanStack Query, Axios |
| **Kiến trúc** | Web không truy cập database trực tiếp, giao tiếp hoàn toàn qua Backend API |
| **Công cụ hỗ trợ** | Google Drive (tiến độ), GitHub (mã nguồn), Postman (kiểm thử), PlantUML (thiết kế CSDL/UML) |

> Riêng công cụ quản lý tiến độ: tài liệu mô tả gốc định hướng dùng Trello; nhóm chỉ có 2 thành viên nên quản lý tiến độ bằng Google Drive thay thế (xem Mục 16).

**Nguyên tắc kiểm soát phạm vi:** Không tự ý thêm chức năng nằm ngoài nghiệp vụ. Mọi chức năng mới phải có cơ sở từ nghiệp vụ, yêu cầu của hệ thống hoặc tiêu chí môn học.

---

## 16. Truy vết và phân loại

| Hạng mục | Nguồn / Căn cứ |
|---|---|
| 3 vai trò: Admin, Chủ trọ, Người thuê | Tài liệu mô tả chi tiết đề tài |
| Actor Trợ lý AI | Tài liệu mô tả chi tiết đề tài |
| Nền tảng Web duy nhất | Tài liệu mô tả chi tiết đề tài |
| Tích hợp AI trợ lý và Matching ở ghép | Tài liệu mô tả chi tiết đề tài |
| Ràng buộc quyền lực của AI (BR-03) | Tài liệu mô tả chi tiết đề tài |
| Công thức tính tiền điện/nước theo chỉ số (BR-14) | Tài liệu mô tả chi tiết đề tài |
| Phân bổ Đồ án 1, 02 thành viên, 2.5 tháng | Tài liệu mô tả chi tiết đề tài |
| Định hướng công nghệ ASP.NET, React, Supabase | Tài liệu mô tả chi tiết đề tài |
| Hiện trạng As-Is (Mục 2.1) | Khảo sát thực tế mô hình cho thuê trọ quy mô nhỏ tại Việt Nam |
| Mục tiêu nghiệp vụ G-01 → G-06 | Suy ra từ các nỗi đau đã xác định ở Mục 2.1 |
| **BP-01 → BP-13** (toàn bộ quy trình) | Phân tích nghiệp vụ dựa trên các đầu mục chức năng của đề tài |
| BP-05 (Đặt lịch xem phòng) | Bổ sung do phát hiện thiếu bước khảo sát thực tế trước khi thuê |
| BP-09 (Gia hạn hợp đồng) | Bổ sung do vòng đời hợp đồng ban đầu không xử lý trường hợp tiếp tục thuê |
| Khái niệm **Tiền cọc** và toàn bộ luồng cọc | Thực tiễn cho thuê trọ: cọc là điều kiện giữ phòng và bảo đảm nghĩa vụ hợp đồng; thanh lý cần đối chứng khoản đã thu |
| Khái niệm **Người ở cùng** | Bổ sung để xử lý mâu thuẫn giữa "số người tối đa" của phòng và mô hình một người đứng tên hợp đồng |
| Khái niệm **Nhật ký hệ thống** | BP-13 yêu cầu Admin đối chứng khi xử lý khiếu nại; cần một nguồn dữ liệu không sửa xóa làm căn cứ |
| Tách trạng thái Phòng thành 2 chiều (khai thác / hiển thị) | Tình trạng khai thác và việc quảng bá tin là hai yếu tố độc lập, thay đổi vì những lý do khác nhau |
| Trạng thái *Đang giữ chỗ* của Phòng | Sửa lỗ hổng phòng vẫn hiển thị "Trống" trong lúc chờ ký hợp đồng |
| Trạng thái *Chờ xác nhận* của Hóa đơn | BR-06b yêu cầu Chủ trọ đối soát minh chứng trước khi hóa đơn được coi là đã trả |
| BR-26 (mã VietQR của Chủ trọ) | Yêu cầu hỗ trợ thanh toán trực tuyến của giảng viên hướng dẫn; tiền chuyển thẳng cho Chủ trọ, không cần tài khoản doanh nghiệp hay cổng thanh toán |
| BR-06 (tự động từ chối yêu cầu trùng phòng) | Sửa lỗi tranh chấp đồng thời trên cùng một phòng |
| BR-12, BR-13 (chốt giá tại thời điểm ký/phát hành) | Ngăn lỗi tính lại hóa đơn cũ khi Chủ trọ đổi giá |
| BR-15 (tính tiền phòng theo tỷ lệ ngày) | Xử lý kỳ thuê đầu và cuối không trọn tháng |
| BR-24, BR-25, Mục 10 (ranh giới đọc dữ liệu) | Giới hạn quyền đọc của AI và của Admin để tránh rò rỉ dữ liệu tài chính và thông tin cá nhân |
| Quyết định "Tin đăng = Phòng đang hiển thị" | Giảm độ phức tạp mô hình dữ liệu và loại bỏ nguy cơ dữ liệu phòng lệch với dữ liệu tin đăng |
| Danh mục sự kiện thông báo (Mục 9) | Cụ thể hóa hạng mục "thông báo" vốn chỉ được nhắc chung chung |
| Nguyên tắc chuyển trạng thái tuyến tính, không nhảy cóc (BR-05b) | Kế thừa nguyên tắc nhất quán trạng thái trong phân tích hệ thống; các chuyển trạng thái lùi hợp lệ được khai báo tường minh tại Mục 8 |
| Phân kỳ Phase 1/2/3 | Đánh giá rủi ro nguồn lực theo CO-01 và CO-02 |
| Chỉ số điện nước lúc bàn giao ghi trong Hợp đồng (BR-14) | Kỳ hóa đơn đầu tiên không có kỳ liền trước để lấy chỉ số cũ; thực tế hai bên chốt số đồng hồ lúc giao phòng |
| Kỳ hóa đơn theo tháng dương lịch, do hệ thống xác định (BR-15, BR-17) | Để Chủ trọ tự chọn ngày kỳ thì có thể lập chồng kỳ và thu tiền phòng hai lần. Phí dịch vụ tháng lẻ cũng tính theo tỷ lệ vì thu trọn tháng với người vào ở cuối tháng là bất hợp lý |
| Điều chỉnh sai sót bằng dòng *Điều chỉnh* ở kỳ sau (BR-16) | Thực tế Chủ trọ cộng hoặc trừ phần chênh lệch vào tháng sau; sửa một hóa đơn cũ khi đã có kỳ sau sẽ làm gãy chuỗi chỉ số của BR-14 |
| Kết chuyển công nợ cũ vào Hóa đơn thanh lý | Tiền cọc trước hết dùng để trừ nợ còn lại; để hóa đơn cũ tồn tại song song với dòng công nợ sẽ tính nợ hai lần |
| Bảng thanh lý đi qua bước gửi — đồng ý — khóa | Thực tế hai bên trao đổi và sửa bảng tới khi thống nhất; Phase 1 chưa có khiếu nại nên cần đường quay lại để Chủ trọ sửa |
| Người thuê hủy sau khi đặt cọc có thể mất cọc (BR-22) | Thực tế người thuê đổi ý sau khi đặt cọc thì mất cọc, vì phòng đã được giữ và gỡ khỏi tìm kiếm cho họ; khớp với vế "Người thuê đơn phương chấm dứt" vốn có trong BR-22 |
| Admin ẩn tin vi phạm (BP-03 A1) dời sang Phase 2 | Phase 1 chưa có kênh báo cáo nên Admin không có căn cứ tìm tin vi phạm; khóa tài khoản Chủ trọ đã gỡ được toàn bộ tin của người đó theo BR-05 |
| Chủ trọ tự chốt bảng thanh lý sau 7 ngày; hoàn tất thanh lý khi người thuê còn nợ | Thực tế người thuê dọn đi rồi không phản hồi hoặc không trả nốt tiền là chuyện thường gặp; nếu phải chờ người thuê thì hợp đồng kẹt ở *Đang thanh lý* và BR-08 khiến phòng không cho thuê lại được |
| Tháng trả phòng không có hóa đơn định kỳ; báo gấp sau khi đã lập thì thanh lý không tính tiền phòng | Quy tắc lập hóa đơn từ ngày 25 có thể đụng tháng trả phòng; thông báo dưới 30 ngày vẫn được chấp nhận, nên người thuê báo gấp đã trả trọn tháng — phù hợp với việc báo trước không đủ |
| Ghi nhận bên gửi thông báo trả phòng; phí phạt chỉ khi Người thuê báo gấp (BR-22) | Không biết ai chấm dứt thì không kiểm được điều kiện phạt. "Chủ trọ chấm dứt phải hoàn toàn bộ cọc" được hiểu là không có phí phạt, vì tiền nợ và hư hỏng vẫn phải trừ |
| Chưa được làm Chủ trọ khi còn hợp đồng hoặc yêu cầu thuê đang mở (BR-01) | Mỗi tài khoản một vai trò: đổi sang Chủ trọ giữa chừng làm người dùng mất quyền báo thanh toán, đồng ý thanh lý trên hợp đồng của chính mình |
| Rút thông báo trả phòng (BP-10 A4) | Thực tế người thuê đổi ý hoặc hai bên thỏa thuận ở tiếp; không có đường quay lại thì hợp đồng buộc phải thanh lý dù không ai muốn |
| Người thuê rút được yêu cầu thuê đã duyệt khi chưa lập hợp đồng (BP-06 A1) | Người thuê đổi ý sau khi được duyệt thì phòng không phải bị giữ vô ích tới hết 72 giờ |
| Khóa Chủ trọ đang có người thuê là giới hạn được chấp nhận ở Phase 1 (BP-01 A2) | Cho tài khoản bị khóa vẫn thao tác được một phần cần cơ chế phân quyền riêng; Phase 1 chọn cảnh báo Admin trước khi khóa |
| Địa chỉ theo đơn vị hành chính 2 cấp (tỉnh/thành, phường/xã) | Từ 01/07/2025 không còn cấp quận/huyện; chọn từ danh mục chính thức để bộ lọc khu vực không lệch vì cách gõ tên khác nhau |
| Phòng phải có ít nhất một ảnh mới được đăng tin | Tin không ảnh gần như vô dụng với người tìm phòng và dễ là tin ảo (G-04) |
| Quản lý tiến độ bằng Google Drive thay cho Trello | Nhóm chỉ có 2 thành viên; một bảng tiến độ chung trên Google Drive đủ dùng, không cần thêm công cụ quản lý công việc riêng |
| Sửa phí dịch vụ của phòng cũng ghi nhật ký (BR-23) | Phí dịch vụ được chốt vào hợp đồng như giá thuê (BR-12) và ảnh hưởng trực tiếp tới tiền người thuê trả |
| Chủ trọ hủy duyệt yêu cầu thuê khi chưa lập hợp đồng (BP-06 A5) | Thực tế người thuê được duyệt rồi không đến hoặc không liên lạc được; không có đường này thì phòng bị giữ vô ích tới hết 72 giờ, hoặc Chủ trọ phải lập hợp đồng rồi hủy cho nhanh |
| Người thuê xem được hợp đồng ở *Nháp* (chỉ đọc) | Hợp đồng quay về *Nháp* sau khi Người thuê yêu cầu chỉnh sửa hoặc Chủ trọ thu hồi; Người thuê đã thấy điều khoản từ trước và cần chỗ để hủy nếu đổi ý |
| Chủ trọ thu hồi hợp đồng đã gửi để sửa (BP-06 A6) | Thực tế gửi xong mới thấy gõ sai giá hoặc ngày; nếu chỉ có cách hủy hợp đồng thì yêu cầu thuê đã kết thúc, Người thuê phải gửi yêu cầu mới và chờ duyệt lại |
| Nhắc bên đang phải thao tác khi hạn giữ chỗ còn dưới 24 giờ (BP-06 A3) | Chỉ nhắc Người thuê là nhắc nhầm người khi Chủ trọ chưa lập hoặc chưa gửi hợp đồng; tệ nhất là Người thuê đã chuyển cọc mà Chủ trọ quên xác nhận, hết 72 giờ hệ thống tự hủy hợp đồng dù tiền đã chuyển |
| Một Người thuê chỉ giữ một phòng tại một thời điểm (BR-28) | Thực tế người tìm phòng gửi yêu cầu nhiều nơi cùng lúc; nếu mấy Chủ trọ cùng duyệt thì một người giữ mấy phòng suốt 72 giờ và các phòng kia mất khách |
| Kiểm tra ngày vào ở và số người ngay khi gửi yêu cầu thuê | Số người vượt sức chứa thì đằng nào cũng không lập được hợp đồng (BR-11); chặn từ đầu để Chủ trọ không duyệt rồi giữ phòng vô ích |

---

*Hết tài liệu*
