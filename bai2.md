# Functional Requirements - Module Authentication

## FR-AUTH-001: Đăng nhập hệ thống (System Login)
- **Mô tả**: Cho phép người dùng (với các vai trò Admin, Staff, Customer) đăng nhập vào hệ thống Shop AI bằng tài khoản đã được đăng ký thông qua Email và Mật khẩu. Hệ thống tiến hành xác thực thông tin định danh, kiểm tra trạng thái tài khoản, phân quyền dựa trên mô hình kiểm soát truy cập dựa trên vai trò (RBAC - Role-Based Access Control) và cấp phát cặp khóa bảo mật (Access Token & Refresh Token) để thiết lập phiên làm việc.
- **Đầu vào**:
  - `Email`: Chuỗi ký tự (String), bắt buộc, tuân thủ định dạng email chuẩn RFC 5322.
  - `Password`: Chuỗi ký tự (String), bắt buộc, tối thiểu 8 ký tự.
- **Đầu ra**:
  - Mã trạng thái phản hồi HTTP (HTTP Status Code: 200 OK, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 423 Locked).
  - Đối với HTTP 200 OK: Một đối tượng JSON chứa:
    - `accessToken`: Chuỗi JWT Token chứa các thông tin (Claims) bao gồm: `sub` (ID người dùng), `email`, `roles` (danh sách vai trò của người dùng), `exp` (thời hạn hết hạn của token - thiết lập 15 phút), và `iat` (thời điểm phát hành).
    - `refreshToken`: Chuỗi mã định danh ngẫu nhiên có độ bảo mật cao (UUIDv4 hoặc JWT riêng biệt), có thời hạn sử dụng dài hơn (thiết lập 7 ngày), được lưu trữ phía Server (Database/Redis) liên kết với ID tài khoản để phục vụ làm mới Access Token.
    - `tokenType`: "Bearer".
    - `user`: Đối tượng chứa thông tin cơ bản: `id`, `email`, `fullName`, `role`.
  - Đối với các lỗi (HTTP 4xx/5xx): Đối tượng JSON chứa mã lỗi hệ thống (`errorCode`) và thông điệp mô tả chi tiết (`message`).
- **Luong chinh**:
  1. Người dùng truy cập vào giao diện Đăng nhập (Login Page) của hệ thống Shop AI.
  2. Người dùng nhập Email và Mật khẩu, sau đó nhấn chọn nút "Đăng nhập".
  3. Client thực hiện kiểm tra định dạng dữ liệu đầu vào. Nếu dữ liệu không hợp lệ (email sai cấu trúc hoặc mật khẩu trống), Client hiển thị lỗi trực tiếp trên giao diện và dừng luồng xử lý trước khi gửi request.
  4. Client gửi yêu cầu HTTP POST đến endpoint `/api/v1/auth/login` kèm theo Email và Mật khẩu trong Request Body (được truyền tải an toàn qua giao thức HTTPS).
  5. Hệ thống Backend nhận yêu cầu, truy vấn cơ sở dữ liệu để tìm kiếm thông tin tài khoản dựa theo Email được cung cấp.
  6. Hệ thống thực hiện kiểm tra trạng thái hoạt động của tài khoản. Tài khoản bắt buộc phải ở trạng thái "Hoạt động" (Active).
  7. Hệ thống thực hiện giải băm mật khẩu người dùng gửi lên và so khớp với mật khẩu đã mã hóa (bằng thuật toán băm an toàn Bcrypt hoặc Argon2) lưu trữ trong cơ sở dữ liệu.
  8. Quá trình so khớp mật khẩu thành công. Hệ thống tiến hành đặt lại (reset) bộ đếm số lần đăng nhập sai liên tiếp (Failed Login Attempts) của tài khoản này về 0.
  9. Hệ thống khởi tạo cặp token xác thực phiên làm việc:
     - **Access Token (JWT)**: Được ký bằng khóa bí mật (Secret Key) phía Server sử dụng thuật toán ký HS256 hoặc RS256, thời hạn hiệu lực (TTL) là 15 phút.
     - **Refresh Token**: Được tạo mới ngẫu nhiên, lưu vào bộ nhớ đệm phân tán Redis (hoặc Database) liên kết với `userId` kèm thời hạn hiệu lực (TTL) là 7 ngày.
  10. Hệ thống phản hồi mã HTTP 200 OK kèm theo dữ liệu Token và thông tin User trong Response Body.
  11. Client nhận phản hồi thành công:
      - Lưu trữ Access Token vào bộ nhớ tạm (Memory/Application State) của ứng dụng để sử dụng đính kèm vào HTTP Header cho các yêu cầu nghiệp vụ tiếp theo.
      - Lưu trữ Refresh Token vào Cookie bảo mật (thiết lập thuộc tính `HttpOnly`, `Secure`, và `SameSite=Strict`).
  12. Client giải mã JWT để lấy thông tin vai trò (`role`) của người dùng và thực hiện điều hướng giao diện:
      - Nếu `role` là "Admin" hoặc "Staff": Điều hướng người dùng đến trang Dashboard quản trị hệ thống.
      - Nếu `role` là "Customer": Điều hướng người dùng đến trang chủ mua sắm (Homepage).
- **Truong hop ngoai le**:
  - **Ex-AUTH-001.1: Thông tin đầu vào sai định dạng**
    - *Điều kiện xảy ra*: Email trống hoặc sai cấu trúc; mật khẩu để trống.
    - *Xử lý của hệ thống*: Hệ thống Backend từ chối xử lý và phản hồi HTTP 400 Bad Request kèm thông điệp: "Thông tin đăng nhập không hợp lệ".
  - **Ex-AUTH-001.2: Email không tồn tại trên hệ thống**
    - *Điều kiện xảy ra*: Hệ thống tìm kiếm không thấy bản ghi nào tương thích với Email được cung cấp trong cơ sở dữ liệu.
    - *Xử lý của hệ thống*: Hệ thống phản hồi HTTP 401 Unauthorized kèm thông điệp chung: "Email hoặc mật khẩu không chính xác" (tránh lỗ hổng rò rỉ thông tin tài khoản - Username Enumeration).
  - **Ex-AUTH-001.3: So khớp mật khẩu thất bại dưới 5 lần liên tiếp**
    - *Điều kiện xảy ra*: Tìm thấy email hoạt động nhưng mật khẩu cung cấp không khớp với mật khẩu mã hóa trong database, đồng thời số lần nhập sai liên tiếp hiện tại nhỏ hơn 5.
    - *Xử lý của hệ thống*:
      1. Hệ thống tăng bộ đếm số lần nhập sai mật khẩu liên tiếp (Failed Login Attempts) của tài khoản trong cơ sở dữ liệu/Redis lên 1 đơn vị.
      2. Hệ thống phản hồi HTTP 401 Unauthorized kèm thông điệp cảnh báo: "Email hoặc mật khẩu không chính xác. Bạn còn [5 - số lần sai] lần thử trước khi tài khoản bị khóa tạm thời".
  - **Ex-AUTH-001.4: Người dùng nhập sai mật khẩu quá 5 lần liên tiếp (RÀNG BUỘC 1)**
    - *Điều kiện xảy ra*: Người dùng nhập sai mật khẩu lần thứ 5 liên tiếp đối với một tài khoản.
    - *Xử lý của hệ thống*:
      1. Hệ thống tự động cập nhật trạng thái hoạt động của tài khoản thành **Tạm khóa (Temporarily Locked)**.
      2. Lưu dấu thời gian khóa (Lockout Timestamp) và thiết lập thời hạn khóa là 15 phút kể từ thời điểm này.
      3. Ghi nhận nhật ký sự kiện bảo mật (Security Log Event) chứa: Email tài khoản, địa chỉ IP yêu cầu, thời gian ghi nhận và mã định danh lý do khóa ("Brute Force Lockout").
      4. Hệ thống tự động kích hoạt tiến trình gửi email cảnh báo bảo mật đến hòm thư của người dùng, thông báo tài khoản bị khóa tạm thời do nhập sai mật khẩu 5 lần, ghi rõ thời điểm tự động mở khóa và hướng dẫn đổi mật khẩu nếu cần.
      5. Hệ thống phản hồi mã HTTP 423 Locked kèm thông điệp lỗi: "Tài khoản của bạn đã tạm thời bị khóa trong 15 phút do nhập sai mật khẩu quá 5 lần liên tiếp. Vui lòng thử lại sau hoặc sử dụng tính năng khôi phục mật khẩu".
      6. Trong vòng 15 phút tiếp theo kể từ thời điểm khóa, bất kỳ yêu cầu đăng nhập nào đối với tài khoản này đều bị hệ thống chặn và từ chối ngay lập tức tại bước kiểm tra trạng thái tài khoản (Bước 6) mà không cần so khớp mật khẩu, đồng thời phản hồi mã HTTP 423 Locked. Sau 15 phút, tài khoản sẽ tự động quay lại trạng thái hoạt động bình thường khi có yêu cầu đăng nhập hợp lệ tiếp theo.
  - **Ex-AUTH-001.5: Tài khoản ở trạng thái khóa (Inactive) cố tình thực hiện đăng nhập (RÀNG BUỘC 3)**
    - *Điều kiện xảy ra*: Tìm thấy tài khoản tương ứng với Email, nhưng trạng thái tài khoản trong cơ sở dữ liệu đang là **Không hoạt động (Inactive)** hoặc **Bị khóa vĩnh viễn (Banned)**.
    - *Xử lý của hệ thống*:
      1. Tại bước kiểm tra trạng thái tài khoản (Bước 6), hệ thống phát hiện trạng thái tài khoản không hợp lệ (không phải là Active).
      2. Hệ thống thực hiện **từ chối yêu cầu đăng nhập ngay lập tức** và kết thúc xử lý tại Backend.
      3. Hệ thống **tuyệt đối không thực hiện** các bước so khớp mật khẩu hay tạo token để bảo vệ tài nguyên hệ thống và ngăn ngừa Brute Force đối với tài khoản bị khóa.
      4. Ghi nhận log hệ thống về nỗ lực truy cập tài khoản không hoạt động.
      5. Hệ thống phản hồi mã HTTP 403 Forbidden kèm theo thông điệp lỗi rõ ràng: "Tài khoản của bạn hiện đang ở trạng thái Không hoạt động hoặc đã bị khóa. Vui lòng liên hệ với Quản trị viên (Admin) để được hỗ trợ kích hoạt lại".

---

## FR-AUTH-002: Đăng xuất hệ thống (System Logout)
- **Mô tả**: Cho phép người dùng đang đăng nhập thực hiện hủy phiên làm việc hiện tại một cách an toàn. Hệ thống sẽ vô hiệu hóa mã xác thực Access Token hiện tại và xóa bỏ Refresh Token tương ứng để tránh nguy cơ tấn công phát lại (Replay Attack) sau khi người dùng thoát ứng dụng.
- **Đầu vào**:
  - Access Token hiện tại đính kèm trong tiêu đề HTTP Header `Authorization: Bearer <access_token>`.
  - Refresh Token đi kèm trong request body hoặc trích xuất từ Cookie HttpOnly.
- **Đầu ra**:
  - Mã trạng thái phản hồi HTTP 200 OK.
  - Xóa bỏ toàn bộ dữ liệu phiên trên Client và điều hướng người dùng.
- **Luong chinh**:
  1. Người dùng click chọn nút "Đăng xuất" trên giao diện điều hướng của ứng dụng Shop AI.
  2. Client gửi yêu cầu HTTP POST đến endpoint đăng xuất `/api/v1/auth/logout`. Yêu cầu chứa Access Token trong Header Authorization và Refresh Token trong Cookie/Request body.
  3. Hệ thống Backend tiếp nhận yêu cầu, giải mã Access Token để xác nhận danh tính người dùng và lấy thời gian hết hạn còn lại (Remaining TTL) của token.
  4. Hệ thống đưa Access Token hiện tại vào danh sách đen (Blacklist) được lưu trữ trong bộ nhớ đệm Redis, với thời gian sống (TTL) của bản ghi bằng đúng thời gian hết hạn còn lại của token. Tất cả các yêu cầu tiếp theo gửi đến Gateway/Backend sử dụng Access Token này đều bị từ chối và phản hồi HTTP 401 Unauthorized.
  5. Hệ thống truy vấn vào DB/Redis và xóa bỏ hoàn toàn bản ghi Refresh Token liên quan đến phiên làm việc này.
  6. Hệ thống phản hồi HTTP 200 OK kèm thông điệp: "Đăng xuất thành công".
  7. Client nhận phản hồi thành công:
     - Xóa Access Token khỏi bộ nhớ tạm (State) của ứng dụng.
     - Xóa Cookie chứa Refresh Token (hoặc xóa khỏi bộ nhớ lưu trữ cục bộ).
  8. Client điều hướng người dùng quay trở lại trang Đăng nhập (`/login`).
- **Truong hop ngoai le**:
  - **Ex-AUTH-002.1: Access Token gửi lên đã hết hạn trước khi đăng xuất**
    - *Điều kiện xảy ra*: Access Token đính kèm trong request đã quá thời hạn sử dụng 15 phút.
    - *Xử lý của hệ thống*: Hệ thống bỏ qua việc xác thực chữ ký hết hạn của Access Token, vẫn tiến hành xóa Refresh Token đi kèm khỏi DB/Redis để đảm bảo giải phóng phiên làm việc, sau đó phản hồi HTTP 200 OK để Client xóa các thông tin xác thực cục bộ và hoàn tất quy trình đăng xuất.

---

## FR-AUTH-003: Làm mới mã xác thực (Token Refresh)
- **Mô tả**: Cung cấp API endpoint cho phép Client gửi Refresh Token hiện tại lên Backend để nhận về một cặp token mới (Access Token và Refresh Token mới) mà không cần người dùng nhập lại thông tin đăng nhập, triển khai kèm cơ chế Refresh Token Rotation (RTR).
- **Đầu vào**:
  - Refresh Token hợp lệ (được gửi tự động qua HttpOnly Cookie hoặc đính kèm trong request body).
- **Đầu ra**:
  - Mã trạng thái HTTP (200 OK hoặc 401 Unauthorized).
  - Đối với HTTP 200 OK: JSON chứa cặp token mới (`accessToken`, `refreshToken`).
- **Luong chinh**:
  1. Client gửi yêu cầu POST đến API endpoint `/api/v1/auth/refresh` mang theo Refresh Token hiện tại.
  2. Backend tiếp nhận yêu cầu, trích xuất Refresh Token và kiểm tra tính hợp lệ của chữ ký và định dạng của token.
  3. Backend truy vấn cơ sở dữ liệu/Redis để đối chiếu:
     - Refresh Token phải tồn tại và khớp với phiên làm việc được lưu trữ của tài khoản.
     - Refresh Token phải còn hạn sử dụng (chưa quá 7 ngày).
  4. Backend kiểm tra trạng thái của tài khoản sở hữu token, đảm bảo trạng thái tài khoản đang là Hoạt động (Active).
  5. Backend thực hiện cơ chế **Refresh Token Rotation (RTR)**:
     - Tạo một Access Token mới (TTL: 15 phút) chứa thông tin vai trò hiện tại của người dùng.
     - Tạo một Refresh Token mới (TTL: 7 ngày).
     - Xóa Refresh Token cũ khỏi cơ sở dữ liệu/Redis (hoặc đánh dấu là đã sử dụng và chuyển vào Blacklist của Refresh Token) để triệt tiêu nguy cơ sử dụng lại.
     - Lưu trữ Refresh Token mới vào DB/Redis liên kết với tài khoản người dùng.
  6. Backend phản hồi HTTP 200 OK kèm theo cặp token mới.
  7. Client nhận phản hồi, cập nhật Access Token vào bộ nhớ tạm ứng dụng và ghi đè Refresh Token mới vào Cookie HttpOnly.
- **Truong hop ngoai le**:
  - **Ex-AUTH-003.1: Thiếu Refresh Token**
    - *Xử lý của hệ thống*: Backend phản hồi HTTP 400 Bad Request kèm mã lỗi `MISSING_REFRESH_TOKEN`.
  - **Ex-AUTH-003.2: Refresh Token không hợp lệ hoặc hết hạn**
    - *Xử lý của hệ thống*: Backend phản hồi HTTP 401 Unauthorized kèm mã lỗi `INVALID_REFRESH_TOKEN` hoặc `EXPIRED_REFRESH_TOKEN`. Client tiến hành xóa sạch dữ liệu token cục bộ và chuyển hướng người dùng về trang đăng nhập.
  - **Ex-AUTH-003.3: Phát hiện tái sử dụng Refresh Token đã bị thu hồi (Token Reuse/Replay Attack)**
    - *Điều kiện xảy ra*: Refresh Token gửi lên có thông tin khớp với một token đã nằm trong danh sách "đã sử dụng" (đã được xoay vòng trước đó).
    - *Xử lý của hệ thống*:
      1. Hệ thống nhận diện đây là hành vi xâm nhập trái phép (kẻ tấn công thu thập được Refresh Token cũ và cố tình gửi lại).
      2. Hệ thống Backend ngay lập tức thu hồi toàn bộ tất cả các phiên làm việc và Refresh Token đang hoạt động liên quan đến tài khoản này trong DB/Redis.
      3. Ghi log cảnh báo bảo mật khẩn cấp (Security Alert Level Critical) ghi nhận nỗ lực replay token.
      4. Hệ thống phản hồi HTTP 401 Unauthorized kèm mã lỗi `COMPROMISED_SESSION`.
      5. Client nhận lỗi, xóa sạch bộ nhớ xác thực cục bộ và buộc người dùng đăng nhập lại từ đầu.

---

## FR-AUTH-004: Xử lý JWT Token hết hạn trong khi phiên làm việc đang diễn ra (Active Session JWT Expiration Handling)
- **Mô tả**: Mô tả chi tiết quy trình phối hợp tự động giữa Client và Backend để xử lý lỗi khi mã Access Token (JWT) hết hạn trong khi người dùng đang thực hiện các thao tác tương tác nghiệp vụ trên ứng dụng Shop AI. Quá trình làm mới token phải được diễn ra ngầm (Silent Refresh) nhằm duy trì trải nghiệm liên tục mà không gây gián đoạn cho người dùng. (RÀNG BUỘC 2)
- **Đầu vào**:
  - Yêu cầu HTTP gửi đến API nghiệp vụ kèm theo Access Token đã hết hạn trong HTTP Header (`Authorization: Bearer <expired_token>`).
  - Refresh Token còn hiệu lực được lưu trữ tại Client.
- **Đầu ra**:
  - HTTP Request nghiệp vụ ban đầu được thực thi và trả về dữ liệu thành công (HTTP 200 OK hoặc mã nghiệp vụ tương ứng).
  - Cặp Access Token mới và Refresh Token mới được cập nhật tự động và trong suốt tại Client.
- **Luong chinh**:
  1. Người dùng đang trong trạng thái đăng nhập và thực hiện thao tác nghiệp vụ trên giao diện ứng dụng (ví dụ: nhấn nút "Thêm vào giỏ hàng").
  2. Client gửi một HTTP Request nghiệp vụ đến hệ thống Backend, đính kèm Access Token hiện tại vào Authorization Header.
  3. Bộ lọc xác thực (Authentication Filter/Middleware) tại Backend nhận request, giải mã, kiểm tra chữ ký và kiểm tra thời gian hết hạn (`exp` claim) của Access Token.
  4. Backend phát hiện chữ ký Access Token hợp lệ nhưng thời hạn của token đã hết (vượt quá 15 phút kể từ thời điểm phát hành).
  5. Backend từ chối xử lý yêu cầu nghiệp vụ, trả về mã trạng thái **HTTP 401 Unauthorized** kèm thông điệp lỗi định dạng JSON trong Response Body: `{"code": "TOKEN_EXPIRED", "message": "Access token has expired"}`.
  6. Bộ phận Interceptor (Bộ chặn HTTP Request) tại Client (ví dụ: Axios Interceptor) bắt được phản hồi HTTP 401 và kiểm tra thấy mã lỗi là `TOKEN_EXPIRED`.
  7. Client tạm dừng (queue) tất cả các HTTP Request nghiệp vụ khác đang chờ gửi đi để tránh gửi dồn dập các request lỗi.
  8. Client tự động gửi một POST Request ngầm (Silent Refresh Request) đến endpoint làm mới `/api/v1/auth/refresh` mang theo Refresh Token hiện đang được lưu trữ.
  9. Backend tiếp nhận yêu cầu refresh và xác thực tính hợp lệ của Refresh Token (tuân theo luồng chính của `FR-AUTH-003`).
  10. Backend xác thực thành công, trả về cặp token mới gồm Access Token mới và Refresh Token mới kèm mã trạng thái HTTP 200 OK.
  11. Client nhận phản hồi từ API refresh:
      - Lưu Access Token mới vào bộ nhớ tạm ứng dụng.
      - Cập nhật Refresh Token mới vào Cookie HttpOnly (hoặc bộ lưu trữ tương ứng).
  12. Client giải phóng hàng đợi, lấy request nghiệp vụ ban đầu (bị lỗi ở Bước 5) ra, thực hiện cập nhật lại tiêu đề Authorization bằng Access Token mới (`Authorization: Bearer <new_access_token>`).
  13. Client thực hiện gửi lại request nghiệp vụ đó lên Backend (Retry Request).
  14. Backend nhận request gửi lại, xác thực Access Token mới thành công, tiến hành xử lý nghiệp vụ bình thường và trả về kết quả thành công cho Client.
  15. Client hiển thị kết quả thao tác nghiệp vụ thành công lên giao diện cho người dùng. Toàn bộ quá trình làm mới token và gửi lại request diễn ra hoàn toàn tự động và ngầm dưới nền, người dùng không gặp bất kỳ thông báo lỗi gián đoạn nào.
- **Truong hop ngoai le**:
  - **Ex-AUTH-004.1: Refresh Token cũng đã hết hạn hoặc không hợp lệ (RÀNG BUỘC 2 - Ngoại lệ)**
    - *Điều kiện xảy ra*: Khi Client gửi yêu cầu làm mới ngầm ở Bước 8, Backend xác thực và trả về HTTP 401 Unauthorized kèm mã lỗi `SESSION_EXPIRED` (do Refresh Token cũng đã hết hạn sử dụng 7 ngày hoặc đã bị thu hồi trước đó).
    - *Xử lý của hệ thống*:
      1. Client nhận phản hồi lỗi `SESSION_EXPIRED`.
      2. Client hủy bỏ hàng đợi các request đang chờ xử lý nghiệp vụ.
      3. Client tiến hành xóa sạch Access Token khỏi bộ nhớ tạm và xóa sạch Refresh Token khỏi Cookie/Bộ lưu trữ cục bộ.
      4. Client hiển thị một thông báo dạng hộp thoại Modal cảnh báo cho người dùng: "Phiên làm việc của bạn đã hết hạn do lâu không tương tác. Vui lòng đăng nhập lại để tiếp tục sử dụng hệ thống Shop AI".
      5. Khi người dùng bấm nút xác nhận (hoặc tự động chuyển hướng sau 3 giây), client thực hiện điều hướng người dùng quay trở lại trang Đăng nhập `/login`.
  - **Ex-AUTH-004.2: Gặp sự cố kết nối mạng trong quá trình Refresh ngầm**
    - *Điều kiện xảy ra*: Client thực hiện gửi Silent Refresh Request ở Bước 8 nhưng gặp lỗi kết nối mạng (mất mạng Internet, không kết nối được server).
    - *Xử lý của hệ thống*:
      1. Client tạm dừng gửi lại request, thực hiện cơ chế thử lại (Retry) tối đa 3 lần với khoảng cách 2 giây giữa các lần thử.
      2. Nếu sau 3 lần thử vẫn thất bại, client hiển thị thông báo lỗi kết nối mạng lên màn hình để người dùng biết: "Lỗi kết nối mạng. Vui lòng kiểm tra lại đường truyền".
      3. Client không thực hiện đăng xuất người dùng ngay lập tức để chờ kết nối mạng được khôi phục, nhằm giữ phiên làm việc khi người dùng có mạng trở lại.
