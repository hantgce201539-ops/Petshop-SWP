# Các lỗi và thay đổi mã nguồn UC12–UC17

Bản này mô tả thay đổi đã triển khai, thay cho danh sách lỗi “chưa sửa” trong báo cáo rà soát trước. Căn cứ chính khi có khác biệt là `PetShop_Database_Final.sql` do người dùng cung cấp; đặc tả nghiệp vụ bổ sung những hành vi DB không quy định. Không đổi schema, enum trạng thái, dependency hoặc lockfile.

## 1. Các vấn đề đã sửa

| UC | Lỗi hoặc thiếu sót ban đầu | Cách sửa |
| --- | --- | --- |
| UC12: Manage Cart Items | Sửa số lượng phải bấm nút; số lượng ≤0 vừa bị xóa rồi tiếp tục được lưu; thêm SKU mới vượt tồn bị tự giảm số lượng; chưa chọn được dòng thanh toán. | Tự lưu sau 250 ms khi nhập hoặc ngay khi change, không có nút cập nhật. Thêm checkbox theo `CartItems.IsSelected`. Từ chối số lượng không hợp lệ trước khi ghi, giữ nguyên dữ liệu đã lưu. Kiểm tra sản phẩm/SKU còn bán, chống tràn số, kiểm tra chủ giỏ. |
| UC13: Cart Detail & Pricing | Thiếu cảnh báo khả dụng, chưa có tóm tắt các dòng được chọn; dễ tính nhầm cả giỏ. | Server trả giá và tổng từng dòng, tiền hàng/phí/tổng của các dòng được chọn. Không chọn dòng nào: danh sách thanh toán rỗng, tiền hàng/phí/tổng =0, nút bị khóa. Hết hàng/ngừng bán/vượt tồn có cảnh báo và chặn đặt; vẫn xóa/bỏ chọn được. |
| UC14: Checkout & Order Placement | Trang checkout liệt kê toàn bộ giỏ nhưng backend mua dòng được chọn; tổng chưa trừ voucher; huy hiệu giỏ luôn về0; VNPay đi thẳng trang thành công; chưa chặn gửi lặp. | Checkout chỉ hiển thị dòng được chọn, báo giá và voucher lấy từ server. Khóa giỏ và SKU, kiểm tra lại dữ liệu trong transaction; gửi lặp không mua lại cùng dòng. Chỉ xóa dòng đã mua, tính lại số giỏ còn lại. Bổ sung URL/IPN/Return VNPay, cấu hình thiếu thì không tạo đơn VNPay. Tạo đơn và sinh URL chung transaction để lỗi URL không để lại đơn/giữ tồn dang dở. |
| UC15: View Order History | Thiếu giao diện lọc/phân trang; tham số trang/ngày sai có thể gây exception. | Lọc trạng thái và khoảng ngày đặt, phân trang, giới hạn kích thước trang và offset JPA. Luôn giữ CustomerId của tài khoản đang đăng nhập. Tham số sai hiển thị thông báo. |
| UC16: Customer Order Details | Thiếu thanh toán, mã voucher, thành tiền từng dòng và lịch sử trạng thái; truy cập đơn khác có thể trả lỗi500; dữ liệu lazy có thể lỗi khi hết transaction. | Dùng snapshot đã lưu, bổ sung thông tin còn thiếu và lịch sử theo thời gian. Không lộ ghi chú nội bộ, chỉ hiển thị nhãn actor phù hợp. Đơn không có hoặc không thuộc khách trả404 chung; tải dữ liệu cần thiết trong service. Trang thành công cũng kiểm tra chủ đơn thay vì chỉ phản chiếu mã trên URL. |
| UC17: Cancel Customer Order | Chỉ xét trạng thái đơn, chưa xét phương thức/thanh toán; hủy VNPay chưa đóng giao dịch; yêu cầu đồng thời có thể cùng hoàn tồn/quota. | Kiểm tra chủ đơn, COD chờ xác nhận/chưa trả tiền hoặc VNPay chờ thanh toán có giao dịch pending và chưa thành công. Customer/Staff/IPN/hết hạn dùng cùng khóa đơn. Hủy cập nhật đơn, thanh toán, tồn, voucher và lịch sử trong transaction; yêu cầu lặp không hoàn lần hai. Lý do giới hạn1000 ký tự, lịch sử không vượt giới hạn DB. |

## 2. Hai yêu cầu cụ thể về giỏ hàng

- Khi đổi số lượng, giao diện tự gửi yêu cầu và cập nhật thành tiền, tóm tắt và huy hiệu giỏ. Nhập nhanh được gộp, yêu cầu ghi chạy tuần tự, phản hồi cũ không ghi đè giá trị mới đang nhập.
- Khi còn thay đổi chưa lưu hoặc trạng thái chưa xác nhận do lỗi mạng, nút thanh toán bị khóa. Nếu server có thể đã ghi trước khi mạng lỗi, giao diện đọc lại giỏ để đối chiếu trước khi cho thanh toán.
- Checkbox lưu lựa chọn trên server, không chỉ lọc giao diện. Backend tạo đơn cũng chỉ đọc các dòng `IsSelected=true` thuộc giỏ của Customer hiện tại.
- Không chọn món nào vẫn thấy khung tóm tắt, không có dòng hàng, tổng0 và không đặt được đơn. Chọn một món thì khung tóm tắt và checkout chỉ có món đó.
- Dòng chưa chọn vẫn nằm trong giỏ sau khi đặt. Thao tác giỏ không trừ hoặc giữ tồn; tồn chỉ bị trừ khi tạo đơn trong giao dịch dữ liệu.
- Giữ mặc định `IsSelected=true` khi thêm SKU mới, phù hợp DB hiện có. Có thể bỏ chọn bất kỳ dòng nào.

## 3. Lỗi CSRF phát hiện trên trình duyệt thật

Spring Security6.3 coi request JWT đến HTML, JS và API như các lần xác thực mới khi cấu hình session theo cách cũ. Request tài nguyên tĩnh có thể xóa CSRF cookie vừa được trang giỏ tạo, khiến POST tự lưu bị từ chối403.

`SecurityConfig` vẫn dùng chính sách stateless, nhưng đặt qua shared object và yêu cầu chiến lược xác thực tường minh. Cách này tránh cơ chế đăng nhập ngầm chạy lại trên mỗi request JWT. Bảo vệ CSRF và quyền theo vai trò vẫn được giữ; frontend gửi token/header đúng tên server cung cấp.

Đã có kiểm thử bằng JWT cookie thật và CSRF cookie/header thật qua HTML → JS → API → POST, không dùng helper giả lập `csrf()` cho luồng này. POST thiếu token vẫn bị chặn.

## 4. Hạn chế thay đổi các UC khác

- Tái sử dụng `CartItems.IsSelected`; giữ endpoint form cũ để các trang sản phẩm/thành phần khác tiếp tục thêm hoặc sửa giỏ.
- Không sửa trang Staff/Admin, chức năng địa chỉ, danh mục, hồ sơ, đánh giá hoặc ví voucher.
- Chỉ bổ sung khóa vào các thao tác Staff vốn ghi cùng đơn, để Customer hủy và Staff xác nhận không ghi đè kết quả của nhau. Giữ tiến trình COD xác nhận → chuẩn bị → giao → hoàn thành và chỉ đánh dấu COD đã trả tiền khi hoàn thành.
- Giữ service API cũ; thêm overload lọc ngày và các phương thức đọc/khóa dùng cho thao tác mới.
- Lượt voucher theo khách được đọc trước khi giữ khóa SKU/voucher, trong khi vẫn khóa giỏ của khách để serialize checkout. Cách này tránh vòng chờ khóa với hủy đơn trên SQL Server; nếu hủy xảy ra đồng thời, kết quả kiểm tra cũ chỉ có thể từ chối thận trọng, không cho vượt giới hạn.
- Tác vụ hết hạn xử lý từng đơn trong transaction riêng, không giữ khóa các đơn trước đó khi chuyển sang đơn tiếp theo.

## 5. Quyết định theo DB

- Giữ `ShippingDistrict` bắt buộc và tên tỉnh/huyện/xã hiện tại vì SQL chưa có ProvinceCode/WardCode. Không chuyển riêng checkout sang mô hình hai cấp làm hỏng hợp đồng địa chỉ/đơn cũ.
- Phí cố định30.000đ theo DB, dùng một hằng chung cho preview và tạo đơn. Không thêm cấu hình phí Admin hoặc API vận chuyển.
- Giữ `Orders.PaymentStatus`: UNPAID/PAID/FAILED/CANCELLED. PENDING/SUCCESS/EXPIRED là trạng thái giao dịch, không thêm chúng vào enum đơn.
- Không dựng lại InventoryTransactions mà SQL đã bỏ. Đơn, dòng đơn và lịch sử trạng thái hiện có tiếp tục là dữ liệu đối chiếu bán/hủy.
- Giữ snapshot giá/địa chỉ/voucher của đơn đã đặt; không lấy giá và địa chỉ mới để thay thế dữ liệu lịch sử.
- Cảnh báo giá đã thay đổi so với lần xem trước cần mốc so sánh mà schema giỏ hiện chưa lưu. Bản sửa hiển thị giá hiện tại và kiểm tra lại giá phía server, chưa bổ sung lịch sử giá giỏ hoặc migration chỉ để có cảnh báo này.

## 6. VNPay Sandbox

Các biến cần cấu hình an toàn ở môi trường chạy: `VNPAY_TMN_CODE`, `VNPAY_HASH_SECRET`, `VNPAY_RETURN_URL`. Không ghi giá trị thật vào repository. Cấu hình IPN URL `/payment/vnpay-ipn` trong portal VNPay; Return URL trỏ `/payment/vnpay-return` tại host ứng dụng.

- Chưa cấu hình đầy đủ: VNPay bị khóa trên UI, request ép chọn VNPay bị từ chối trước khi tạo đơn; COD vẫn dùng được.
- URL dùng HMAC-SHA512; IPN kiểm tra chữ ký, mã merchant, tham chiếu, số tiền và cả hai mã kết quả. Callback bị lặp, giả mạo, trùng tham số hoặc đến sau hủy không xác nhận/hoàn tồn lần nữa.
- IPN thành công mới xác nhận đơn. Return chỉ đọc, kiểm tra quyền chủ đơn và chuyển về chi tiết; không tự cập nhật thanh toán.
- Callback thành công đến muộn sau khi đơn đóng chỉ ghi dấu đối soát, không hồi sinh đơn hoặc tự refund.
- Đơn chờ hết hạn được quét mỗi phút. Giá trị lẻ đồng không được âm thầm làm tròn khi gửi VNPay; trả thông báo chọn COD và rollback tạo đơn nếu sinh URL thất bại.

## 7. Kiểm chứng

Đã chạy Maven `verify`: **118 tests,0 failures,0 errors,0 skipped**, đóng gói JAR thành công. Gồm11 kiểm thử sẵn có và107 kiểm thử mới về giỏ, API/CSRF, checkout, đơn hàng, DTO và VNPay.

Có kiểm thử đa luồng bằng transaction thật trên H2: checkout trùng, hai lần hủy, hủy so với Staff xác nhận, tác vụ hết hạn so với hủy trên các đơn chung SKU. Có kiểm thử lỗi ghi sau khi giữ voucher/trừ SKU đầu tiên để xác nhận rollback.

Chromium chạy với ứng dụng thật: đăng nhập JWT,3 dòng giỏ, bỏ chọn hết, chọn một dòng, đổi số lượng tự động/gõ nhanh, phục hồi số lượng không hợp lệ, thử/gỡ voucher qua CSRF thật, đặt COD và giữ2 dòng chưa chọn, xem chi tiết và hủy đơn. Không có lỗi JavaScript trong luồng này. Bộ harness riêng cho giỏ đạt29 assertions về thứ tự phản hồi, lỗi400, mất kết nối sau commit và retry.

Lệnh kiểm tra tại môi trường này:

```bash
source /workspace/.cloud-tools/activate.sh
cd /workspace/Petshop-SWP
mvn -B -ntp -s /workspace/.cloud-tools/settings.xml \
  -Dmaven.repo.local=/workspace/.cloud-tools/m2 \
  -Dspring.jpa.properties.hibernate.hbm2ddl.create_namespaces=true verify
```

Kiểm thử integration dùng profile test/H2; tùy chọn tạo schema dbo chỉ dành cho H2. Giữ `ddl-auto=validate` khi chạy với SQL Server.

Chưa kiểm thử kết nối SQL Server thật: tải image SQL Server bị registry trả `Forbidden`. Chưa thực hiện giao dịch merchant VNPay Sandbox thật vì chưa có cấu hình merchant/callback công khai. Các giới hạn này không được coi là đã xác thực bởi test mô phỏng.
