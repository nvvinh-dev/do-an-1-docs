# Đặc tả Yêu cầu Chức năng — Phase 1

Tài liệu chính thức về yêu cầu chức năng của hệ thống SmartRent.

**Phạm vi:** Phase 1 theo phân kỳ trong [README](../README.md).

**Cách đọc:** mỗi yêu cầu được viết ở dạng **kiểm chứng được** — đọc xong phải biết ngay cần thử gì để kết luận hệ thống đạt hay không đạt. Cột *Nguồn* trỏ về quy trình nghiệp vụ (BP) hoặc quy tắc nghiệp vụ (BR) trong tài liệu phân tích. Cột *Hiện thực* trỏ về [thiết kế API](api-design.md) và [thiết kế cơ sở dữ liệu](database-design.md).

Tài liệu này **không** định nghĩa quy tắc nghiệp vụ mới. Mọi FR đều truy ngược được về một BP hoặc BR đã có.

---

## 1. Tài khoản, xác thực và phân quyền

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-01** | Khách truy cập đăng ký được tài khoản bằng email và mật khẩu; tài khoản mới **luôn** nhận vai trò Người thuê | BP-01 | `POST /auth/register` · `users`, `user_roles` |
| **FR-02** | Hệ thống từ chối đăng ký khi email đã tồn tại | BP-01 | `users.email` UNIQUE |
| **FR-03** | Người dùng đăng nhập bằng email và mật khẩu, nhận về JWT kèm danh sách vai trò | BP-01 | `POST /auth/login` |
| **FR-04** | Tài khoản đang bị khóa không đăng nhập được và nhận thông báo kèm lý do khóa | BP-01 | `users.is_locked`, `lock_reason` |
| **FR-05** | Người dùng đổi được mật khẩu khi đã đăng nhập, và đặt lại được mật khẩu khi quên | BP-01 | `POST /auth/change-password`, `/auth/reset-password` |
| **FR-06** | Người thuê nộp được hồ sơ đăng ký làm Chủ trọ gồm số CCCD, ảnh hai mặt CCCD và giấy tờ chứng minh quyền sở hữu, và tài khoản nộp phải có số điện thoại; thiếu bất kỳ mục nào thì hệ thống từ chối. Ảnh giấy tờ phải do chính người nộp tải lên | BR-01 | `POST /landlord-applications` · `landlord_applications`, `users.phone_number` |
| **FR-07** | Hệ thống từ chối nộp hồ sơ mới khi người dùng đã có một hồ sơ đang chờ duyệt | BP-01 | `landlord_applications.status` |
| **FR-08** | Admin duyệt hồ sơ thì tài khoản người nộp chuyển từ vai trò Người thuê sang vai trò Chủ trọ và số điện thoại được ghi nhận là đã xác thực; Admin từ chối thì **bắt buộc** nhập lý do và người nộp được nộp lại hồ sơ mới | BP-01, BR-01 | `POST /admin/landlord-applications/{id}/approve`, `/reject` · `user_roles`, `users.phone_number_confirmed` |
| **FR-09** | Admin khóa và mở khóa được tài khoản; khi khóa **bắt buộc** nhập lý do | BP-01 | `POST /admin/users/{id}/lock`, `/unlock` |
| **FR-10** | Ảnh CCCD và giấy tờ sở hữu chỉ hiển thị cho Admin trong màn hình duyệt hồ sơ, không xuất hiện ở bất kỳ response nào khác — kể cả response tải file của chính người nộp | QR-04 | `GET /admin/landlord-applications/{id}` |
| **FR-83** | Người dùng đã đăng nhập xem và cập nhật được họ tên, số điện thoại của mình; đổi số điện thoại thì trạng thái đã xác thực của số đó bị bỏ | BP-01, BR-01 | `GET /auth/me`, `PUT /auth/me` · `users.phone_number_confirmed` |

---

## 2. Khu trọ và phòng trọ

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-11** | Chủ trọ tạo được nhiều khu trọ; mỗi khu trọ thuộc về đúng một Chủ trọ | BR-02 | `POST /properties` · `properties.landlord_user_id` |
| **FR-12** | Chủ trọ thêm được phòng vào khu trọ của mình với đầy đủ: mã phòng, diện tích, số người tối đa, giá thuê, đơn giá điện, đơn giá nước và các khoản phí dịch vụ cố định | BP-02 | `POST /properties/{id}/rooms` · `rooms`, `room_service_fees` |
| **FR-13** | Chủ trọ không thao tác được trên khu trọ hoặc phòng không thuộc mình | BR-04 | Kiểm tra sở hữu ở mọi endpoint |
| **FR-14** | Chủ trọ sửa được giá thuê và đơn giá điện nước của phòng; thay đổi này **không** làm đổi giá của hợp đồng đang hiệu lực và hóa đơn đã phát hành | BR-12, BR-13 | `PUT /rooms/{id}` · `contracts`, `invoices` lưu bản sao giá |
| **FR-15** | Chủ trọ bật và tắt được trạng thái hiển thị của phòng; phòng mới tạo ở trạng thái Đã ẩn bởi Chủ trọ cho tới khi Chủ trọ bật | BP-03 | `PATCH /rooms/{id}/visibility` · `rooms.visibility_status` |
| **FR-16** | Phòng bị Admin ẩn thì Chủ trọ **không** tự bật hiển thị lại được | BP-03 A1 | `rooms.visibility_status = DaAnBoiAdmin` |
| **FR-17** | Hệ thống tự gỡ phòng khỏi kết quả tìm kiếm khi phòng chuyển sang Đang giữ chỗ, Đang thuê hoặc Bảo trì | BP-03 A2, BR-05 | `rooms.occupancy_status` |
| **FR-18** | Hệ thống từ chối chuyển phòng về trạng thái Trống khi hợp đồng hiện tại của phòng chưa ở Đã thanh lý hoặc Đã hủy | BR-08 | `PATCH /rooms/{id}/occupancy-status` |
| **FR-89** | Chủ trọ chỉ tự chuyển được trạng thái khai thác của phòng giữa Trống và Bảo trì; các trạng thái khai thác còn lại do hệ thống chuyển theo yêu cầu thuê, hợp đồng và thanh lý, ngoài thao tác lưu trữ | BP-02 | `PATCH /rooms/{id}/occupancy-status` |
| **FR-19** | Phòng hoặc khu trọ đã từng phát sinh hợp đồng hay hóa đơn **không** xóa được, chỉ chuyển sang trạng thái Lưu trữ | BR-09 | Không có endpoint `DELETE` |
| **FR-20** | Hệ thống từ chối lưu trữ khu trọ khi còn phòng ở trạng thái Đang giữ chỗ hoặc Đang thuê | BR-10 | `POST /properties/{id}/archive` |

---

## 3. Tìm kiếm phòng

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-21** | Người dùng chưa đăng nhập tìm kiếm và xem được chi tiết phòng đang cho thuê | BP-04 | `GET /rooms/search`, `/rooms/{id}/public` |
| **FR-22** | Bộ lọc hỗ trợ: khoảng giá, khoảng diện tích, khu vực, tiện ích và số người tối đa | BP-04 | Tham số của `/rooms/search` |
| **FR-23** | Kết quả tìm kiếm **chỉ** chứa phòng thỏa mãn đồng thời: trạng thái khai thác Trống, trạng thái hiển thị Đang hiển thị, khu trọ đang khai thác, và Chủ trọ không bị khóa | BR-05 | Điều kiện truy vấn |
| **FR-24** | Kết quả tìm kiếm **không** chứa thông tin liên hệ của Chủ trọ | QR-07 | Response của `/rooms/search` |

---

## 4. Yêu cầu thuê và hợp đồng

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-25** | Người thuê gửi được yêu cầu thuê cho một phòng đang ở trạng thái Trống, kèm ngày dự kiến vào ở và số người dự kiến | BP-06 | `POST /rooms/{id}/rental-requests` |
| **FR-26** | Hệ thống từ chối yêu cầu thuê cho phòng không ở trạng thái Trống | BP-06 | `rooms.occupancy_status` |
| **FR-85** | Hệ thống từ chối yêu cầu thuê mới khi người thuê đã có một yêu cầu đang chờ duyệt cho cùng phòng | BR-27 | Unique index có điều kiện trên `rental_requests` |
| **FR-27** | Người thuê rút được yêu cầu của mình khi Chủ trọ chưa xử lý | BP-06 A1 | `POST /rental-requests/{id}/cancel` |
| **FR-28** | Yêu cầu thuê không được xử lý trong 7 ngày tự chuyển sang Hết hạn và phòng không bị giữ chỗ | BP-06 A2 | Tác vụ định kỳ · `rental_requests.status` |
| **FR-29** | Chủ trọ từ chối yêu cầu thuê thì **bắt buộc** nhập lý do | BP-06 | `POST /rental-requests/{id}/reject` |
| **FR-30** | Khi một yêu cầu thuê được duyệt, hệ thống **đồng thời** chuyển phòng sang Đang giữ chỗ và tự từ chối toàn bộ yêu cầu khác đang chờ duyệt của cùng phòng, kèm lý do do hệ thống sinh | BR-06 | `POST /rental-requests/{id}/approve` chạy trong transaction |
| **FR-31** | Chủ trọ lập được hợp đồng từ một yêu cầu đã duyệt, trong đó chốt cứng: giá thuê, đơn giá điện, đơn giá nước, phí dịch vụ, số tiền cọc, ngày bắt đầu, ngày kết thúc và hạn thanh toán (số ngày kể từ khi phát hành hóa đơn). Lập xong thì yêu cầu thuê chuyển sang Đã lập hợp đồng | BR-12, BR-21 | `POST /contracts` · `contracts`, `contract_service_fees` |
| **FR-90** | Chủ trọ nhập chỉ số điện, nước lúc bàn giao khi lập hợp đồng, hệ thống điền sẵn chỉ số cuối đã ghi nhận của phòng; người thuê thấy hai chỉ số này khi xác nhận điều khoản. Hợp đồng đã hiệu lực mà chưa có hóa đơn nào (không tính hóa đơn đã hủy) thì Chủ trọ vẫn sửa được hai chỉ số này; mỗi lần sửa ghi nhật ký và thông báo cho người thuê | BR-14, BR-23 | `POST /contracts`, `PATCH /contracts/{id}/initial-meter-readings` · `contracts.initial_*_index` |
| **FR-32** | Mọi hợp đồng đều ghi nhận số tiền cọc; giá trị 0 được chấp nhận nếu hai bên thỏa thuận không cọc | BR-21 | `contracts.deposit_amount` NOT NULL |
| **FR-33** | Hệ thống từ chối lập hợp đồng khi tổng số người ở vượt quá số người tối đa của phòng | BR-11 | `contract_occupants`, `rooms.max_occupants` |
| **FR-34** | Hợp đồng **chỉ** chuyển sang Đang hiệu lực khi người thuê đã xác nhận điều khoản **và** Chủ trọ đã xác nhận nhận đủ cọc | BR-21 | `POST /contracts/{id}/confirm`, `/deposit/confirm` |
| **FR-84** | Người thuê yêu cầu chỉnh sửa được hợp đồng đang chờ mình xác nhận và **bắt buộc** nhập lý do; hợp đồng quay về Nháp và Chủ trọ nhận thông báo kèm lý do | BP-06 | `POST /contracts/{id}/request-changes` |
| **FR-35** | Khi hợp đồng có hiệu lực, hệ thống tự chuyển phòng sang Đang thuê | BP-06 | `rooms.occupancy_status` |
| **FR-36** | Quá 3 ngày kể từ khi Chủ trọ duyệt yêu cầu thuê mà hợp đồng chưa có hiệu lực thì hợp đồng (nếu đã lập) tự hủy, yêu cầu thuê chưa được lập hợp đồng chuyển sang Hết hạn, phòng trở lại Trống và hiển thị lại | BP-06 A3 | Tác vụ định kỳ · `rental_requests.processed_at` |
| **FR-37** | Tại một thời điểm, một phòng **chỉ** có tối đa một hợp đồng ở trạng thái Đang hiệu lực, Sắp hết hạn hoặc Đang thanh lý | BR-07 | Unique index có điều kiện trên `contracts.room_id` |

---

## 5. Hóa đơn và thanh toán

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-38** | Khi Chủ trọ chốt số, hệ thống **tự điền** chỉ số cũ bằng chỉ số mới của kỳ liền trước, kỳ đầu tiên lấy chỉ số đầu ghi trong hợp đồng; Chủ trọ chỉ nhập chỉ số mới | BR-14 | `POST /contracts/{id}/invoices` |
| **FR-39** | Hệ thống **từ chối** lưu khi chỉ số mới nhỏ hơn chỉ số cũ | BR-14 | Ràng buộc `CHECK` trên `invoices` |
| **FR-40** | Tiền điện và tiền nước được tính bằng (chỉ số mới − chỉ số cũ) × đơn giá **đã chốt trong hợp đồng**, không dùng đơn giá hiện tại của phòng | BR-13, BR-14 | `invoices.electricity_unit_price`, `water_unit_price` |
| **FR-41** | Tổng hóa đơn gồm: tiền phòng + tiền điện + tiền nước + phí dịch vụ cố định + khoản điều chỉnh nếu có | BP-07 | `invoices.total_amount`, `invoice_lines` |
| **FR-42** | Kỳ hóa đơn đầu tiên và kỳ cuối không trọn tháng thì tiền phòng và phí dịch vụ tính theo tỷ lệ số ngày thực ở trên số ngày của tháng, tính cả ngày vào ở và ngày trả phòng; mọi khoản làm tròn đến đồng | BR-15 | `invoices.rent_amount`, `service_fee_amount` |
| **FR-43** | Kỳ hóa đơn là một tháng dương lịch do hệ thống xác định, Chủ trọ không gửi ngày đầu và ngày cuối kỳ; mỗi hợp đồng **chỉ** có một hóa đơn định kỳ chưa hủy cho mỗi kỳ | BR-17 | Unique index trên `contract_id` + kỳ, bỏ qua hóa đơn đã hủy |
| **FR-91** | Chủ trọ lập hóa đơn định kỳ theo đúng thứ tự tháng, cho tháng đã kết thúc hoặc cho tháng hiện tại từ ngày 25 trở đi; kỳ đầu tiên tính từ ngày bắt đầu hợp đồng tới cuối tháng đó | BP-07, BR-15 | `POST /contracts/{id}/invoices` |
| **FR-44** | Người thuê xem được đầy đủ chỉ số cũ, chỉ số mới, đơn giá áp dụng và cách tính của từng hóa đơn | QR-01 | `GET /invoices/{id}` |
| **FR-45** | Người thuê báo đã thanh toán cho hóa đơn ở Chưa thanh toán, Thanh toán một phần hoặc Quá hạn, và **bắt buộc** đính kèm minh chứng; thiếu minh chứng thì hệ thống từ chối | BR-06b | `POST /invoices/{id}/payment-reports` |
| **FR-46** | **Chỉ** Chủ trọ sở hữu mới xác nhận được một hóa đơn đã thu; mọi vai trò khác bị từ chối, kể cả Admin | BR-06b, BR-24 | `POST /invoices/{id}/payment-reports/{reportId}/confirm` |
| **FR-47** | Chủ trọ xác nhận số tiền thu được mà tổng đã thu vẫn nhỏ hơn tổng hóa đơn thì hóa đơn chuyển sang Thanh toán một phần — hoặc Quá hạn nếu đã qua hạn thanh toán — và phần còn lại vẫn được theo dõi là công nợ | BP-07 A2 | `invoices.paid_amount`, `status` |
| **FR-48** | Chủ trọ từ chối xác nhận thanh toán thì **bắt buộc** nhập lý do; hóa đơn quay về Quá hạn nếu đã qua hạn thanh toán, về Thanh toán một phần nếu đã thu được một phần, còn lại về Chưa thanh toán | BP-07 | `POST /invoices/{id}/payment-reports/{reportId}/reject` |
| **FR-49** | Chủ trọ chỉ sửa hoặc hủy được hóa đơn định kỳ **mới nhất** (không tính hóa đơn đã hủy) của hợp đồng khi hóa đơn đó chưa thanh toán; hóa đơn Đã thanh toán **không** sửa được. Sai sót ở hóa đơn không còn sửa được điều chỉnh bằng một dòng Điều chỉnh có mô tả và tham chiếu tới hóa đơn gốc, ở hóa đơn kỳ kế tiếp hoặc hóa đơn thanh lý | BR-14, BR-16 | `invoice_lines.related_invoice_id` |
| **FR-50** | Chủ trọ sửa hóa đơn chưa thanh toán thì hệ thống ghi nhật ký giá trị cũ và mới, đồng thời thông báo cho người thuê | BP-07 A3, BR-23 | `PUT /invoices/{id}` · `audit_logs` |
| **FR-51** | Hóa đơn chưa trả đủ (Chưa thanh toán hoặc Thanh toán một phần) mà qua hạn thanh toán được hệ thống tự gắn cờ Quá hạn và gửi nhắc nhở cho cả hai bên | BP-07 A1 | Tác vụ định kỳ · `invoices.due_date` |
| **FR-52** | Hủy hóa đơn **chỉ** áp dụng cho hóa đơn mới nhất của hợp đồng, chưa thanh toán, và **bắt buộc** nhập lý do | BP-07 A4 | `POST /invoices/{id}/cancel` |
| **FR-77** | Chủ trọ khai báo và cập nhật được tài khoản ngân hàng nhận tiền gồm ngân hàng, số tài khoản và tên chủ tài khoản; thiếu một trong ba mục thì hệ thống từ chối. Việc khai báo **không** bắt buộc. Mỗi lần khai báo hoặc sửa đều được ghi nhật ký với giá trị cũ và mới | BR-26, BR-23 | `PUT /landlord/bank-account` · `users.bank_*` · `audit_logs` |
| **FR-78** | Người thuê đứng tên thấy mã VietQR trên hóa đơn ở Chưa thanh toán, Quá hạn hoặc Thanh toán một phần, với số tiền bằng **phần còn phải trả** (tổng hóa đơn trừ số đã được xác nhận thu). Áp dụng cho cả hóa đơn thanh lý có số dư dương | BR-26 | `GET /invoices/{id}` · trường `paymentQr` |
| **FR-79** | Người thuê đứng tên thấy mã VietQR trên hợp đồng ở Chờ nhận cọc, với số tiền bằng tiền cọc; tiền cọc bằng 0 thì không hiển thị mã | BR-26 | `GET /contracts/{id}` · trường `paymentQr` |
| **FR-80** | Số tiền và nội dung chuyển khoản trong mã VietQR do **server** tính; Chủ trọ chưa khai báo tài khoản thì hệ thống không hiển thị mã | BR-26 | `paymentQr` là `null` |
| **FR-81** | Quét mã VietQR hay chuyển khoản **không** làm thay đổi trạng thái hợp đồng hoặc hóa đơn; tiền chỉ được ghi nhận khi Chủ trọ xác nhận nhận cọc hoặc xác nhận thanh toán | BR-26, BR-06b | Không có endpoint nào nhận kết quả chuyển khoản |

---

## 6. Chấm dứt hợp đồng và tất toán tiền cọc

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-53** | Một trong hai bên gửi được thông báo trả phòng kèm ngày trả dự kiến; hợp đồng chuyển sang Đang thanh lý | BP-10 | `POST /contracts/{id}/move-out-notice` |
| **FR-87** | Thông báo trả phòng gửi trước ít hơn 30 ngày vẫn được chấp nhận; tổng các dòng phí phạt trong hóa đơn thanh lý không vượt số tiền cọc của hợp đồng | BP-10, BR-22 | `POST /contracts/{id}/move-out-notice`, `/settlement-invoice` |
| **FR-88** | Hợp đồng qua ngày kết thúc mà chưa có thông báo trả phòng thì vẫn hiệu lực theo điều khoản đã chốt và hóa đơn định kỳ tiếp tục được lập, cho tới khi một bên gửi thông báo trả phòng | BP-09, BP-10 | `contracts.status`, `POST /contracts/{id}/invoices` |
| **FR-54** | Chủ trọ chốt được chỉ số điện nước lần cuối và lập hóa đơn thanh lý | BP-10 | `POST /contracts/{id}/settlement-invoice` |
| **FR-93** | Hợp đồng ở Đang thanh lý vẫn lập được hóa đơn định kỳ cho các tháng trọn trước ngày trả phòng. Hóa đơn thanh lý tính từ sau kỳ định kỳ cuối cùng đến ngày trả phòng, và khoảng này phải nằm trong một tháng; nếu không, hệ thống từ chối lập hóa đơn thanh lý | BP-10, BR-15 | `POST /contracts/{id}/settlement-invoice` |
| **FR-55** | Hóa đơn thanh lý hiển thị từng khoản mục riêng biệt: tiền phòng kỳ cuối, điện nước kỳ cuối, phí dịch vụ, công nợ kỳ trước, bồi thường hư hỏng, phí phạt, và khoản trừ tiền cọc. Dòng công nợ kỳ trước và dòng trừ tiền cọc do hệ thống tự thêm; Chủ trọ không gửi hai loại dòng này | BP-10 | `invoice_lines.category` |
| **FR-92** | Khi lập hóa đơn thanh lý, mỗi hóa đơn còn thiếu tiền của hợp đồng trở thành một dòng công nợ bằng phần còn phải trả, có tham chiếu tới hóa đơn gốc; hóa đơn gốc chuyển sang Đã chuyển vào thanh lý — không bị nhắc quá hạn và không nhận báo thanh toán riêng. Hệ thống từ chối lập hóa đơn thanh lý khi hợp đồng còn lượt báo thanh toán chờ xác nhận | BP-10, BR-22 | `POST /contracts/{id}/settlement-invoice` · `invoice_lines.related_invoice_id` |
| **FR-56** | **Mọi** khoản khấu trừ tiền cọc phải là một dòng riêng có mô tả lý do; hệ thống từ chối khoản khấu trừ không có mô tả | BR-22 | `invoice_lines.description` NOT NULL |
| **FR-57** | Chủ trọ gửi bảng thanh lý cho người thuê; người thuê **Đồng ý**, hoặc **Chưa đồng ý** kèm lý do (**bắt buộc**) — bảng quay về Nháp để Chủ trọ sửa và gửi lại, Chủ trọ nhận thông báo kèm lý do. Sau khi người thuê đồng ý, bảng bị khóa: số dư dương thành khoản chờ thanh toán, số dư âm chờ Chủ trọ hoàn cọc | BP-10 | `POST /contracts/{id}/settlement-invoice/send`, `/confirm`, `/request-changes` |
| **FR-58** | Số dư cuối cùng âm nghĩa là Chủ trọ phải hoàn lại phần cọc dư; hệ thống hiển thị rõ chiều của số dư | BP-10 | `invoices.total_amount` |
| **FR-59** | Sau khi hoàn tất thanh lý, hợp đồng chuyển sang Đã thanh lý và phòng chuyển sang Bảo trì hoặc Trống theo lựa chọn của Chủ trọ | BP-10 | `POST /contracts/{id}/settlement/complete` |
| **FR-76** | Hai bên đều hủy được hợp đồng chưa tới ngày bắt đầu, **bắt buộc** nhập lý do; hệ thống **không** lập hóa đơn thanh lý và phòng trở lại Trống ngay. Nếu cọc đã nộp: Chủ trọ hủy thì số tiền hoàn bằng **toàn bộ** cọc; Người thuê hủy thì Chủ trọ nhập số tiền hoàn từ 0 tới bằng tiền cọc, số hoàn nhỏ hơn tiền cọc thì **bắt buộc** ghi lý do giữ lại | BP-10 A3, BR-22 | `POST /contracts/{id}/cancel`, `/deposit/refund` · `contracts.cancelled_by_user_id` |
| **FR-86** | Chỉ Chủ trọ ghi nhận được việc hoàn cọc, gồm ngày hoàn và hình thức hoàn. Số tiền hoàn do hệ thống tính — bằng phần cọc dư khi hóa đơn thanh lý có số dư âm, bằng toàn bộ cọc khi Chủ trọ hủy hợp đồng — trừ trường hợp Người thuê hủy ở FR-76. Người thuê xem được thông tin hoàn cọc trên hợp đồng; hợp đồng đã hủy mà chưa ghi nhận hoàn cọc hiển thị là chờ hoàn cọc. Thao tác được ghi nhật ký | BP-10, BR-22, BR-23 | `POST /contracts/{id}/deposit/refund`, `/settlement/complete` · `contracts.deposit_refund*` · `audit_logs` |

---

## 7. Thông báo, nhật ký và dashboard

| Mã | Yêu cầu | Nguồn | Hiện thực |
|---|---|---|---|
| **FR-60** | Hệ thống gửi thông báo trong ứng dụng cho các sự kiện mức Cao thuộc phạm vi Phase 1 | Danh mục sự kiện thông báo | `notifications` |
| **FR-61** | Người dùng xem được danh sách thông báo của mình, số lượng chưa đọc, và đánh dấu được đã đọc | BP-07, BP-06 | `GET /notifications` |
| **FR-62** | Hệ thống ghi nhật ký cho **mọi** thao tác thuộc danh sách BR-23, gồm người thực hiện, thời điểm, giá trị trước và sau | BR-23 | `audit_logs` |
| **FR-63** | Nhật ký hệ thống **không** sửa và **không** xóa được bằng bất kỳ chức năng nào, kể cả với vai trò Admin | QR-03 | Chỉ có endpoint đọc |
| **FR-64** | Admin tra cứu được nhật ký theo đối tượng, người thực hiện và khoảng thời gian | BP-01 | `GET /admin/audit-logs` |
| **FR-65** | Admin xem được dashboard tổng quan: tổng người dùng, chủ trọ, người thuê, khu trọ, phòng và số hồ sơ chờ duyệt | BP-01 | `GET /dashboard/admin` |
| **FR-66** | Chủ trọ xem được dashboard: số phòng trống và đang thuê, hóa đơn chưa thu, doanh thu theo tháng | BP-02, BP-07 | `GET /dashboard/landlord` |
| **FR-67** | Doanh thu trên dashboard **chỉ** tính các khoản đã được Chủ trọ xác nhận thu | BP-07 | `invoices.paid_amount` |
| **FR-68** | Người thuê xem được dashboard: hợp đồng hiện tại, hóa đơn chưa thanh toán và tổng đã thanh toán | BP-07 | `GET /dashboard/tenant` |

---

## 8. Yêu cầu chức năng về an toàn

Các yêu cầu dưới đây là bắt buộc và kiểm chứng được như mọi FR khác. Chi tiết thiết kế nằm ở [Thiết kế An toàn](security-design.md).

| Mã | Yêu cầu | Nguồn |
|---|---|---|
| **FR-69** | Mọi request tới một tài nguyên cụ thể đều được backend kiểm tra quyền sở hữu trước khi thực hiện; thay đổi id trên URL không truy cập được dữ liệu của người khác | BR-04 |
| **FR-70** | Danh tính và vai trò của người thao tác luôn lấy từ token; giá trị vai trò gửi kèm trong body hoặc query bị bỏ qua | Security design 5.2 |
| **FR-71** | Người thuê không xem được hợp đồng, hóa đơn của người thuê khác; Chủ trọ không xem được dữ liệu của khu trọ không thuộc mình | BR-04 |
| **FR-72** | Admin không có chức năng đọc hợp đồng và hóa đơn của người dùng trong Phase 1 | BR-24 |
| **FR-73** | Đăng nhập sai quá 5 lần trong 15 phút, tính theo cặp email + địa chỉ IP, bị tạm chặn; các nhóm endpoint còn lại tuân theo ngưỡng rate limiting đã quy định | Security design 6 |
| **FR-74** | Số điện thoại của bên còn lại chỉ hiển thị khi giữa hai bên có yêu cầu thuê đã được Chủ trọ duyệt, hoặc có hợp đồng chưa kết thúc (chưa Đã thanh lý hoặc Đã hủy) | QR-07 |
| **FR-82** | Tài khoản ngân hàng của Chủ trọ chỉ xuất hiện với chính Chủ trọ đó và trong mã VietQR gửi cho Người thuê đứng tên hợp đồng của Chủ trọ; không xuất hiện trong kết quả tìm kiếm, chi tiết phòng công khai hay bất kỳ response nào khác | BR-26, QR-07 |
| **FR-75** | Mọi thao tác không thể hoàn tác — hủy hợp đồng, xác nhận thanh lý, khóa tài khoản — đều có bước xác nhận rõ ràng trước khi thực hiện | QR-06 |
