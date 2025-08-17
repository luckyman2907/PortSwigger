# OAuth 2.0 authentication vulnerabilities

## OAuth là gì?

Trước khi bắt đầu, có bao giờ bạn thắc mắc rằng tại sao [Gitlab](https://gitlab.com) lại cho phép *Sign in* bằng các website/framework khác (Google, Github, Twitter...)?

![Ví dụ về OAuth](https://github.com/nh4ttruong/portswigger/blob/main/OAuth-2.0-authentication-vulnerabilities/oauth-example.jpg?raw=true)

Từ lúc mình biết đến chức năng **quick sign in** này thì mình có thắc mắc như vậy và cái đó chính là OAuth/OAuth2.0 trên các website.

Hiểu đơn giản rằng, OAuth sẽ cho phép một website sử dụng chức năng ủy quyền tài khoản của user với tài khoản ở một website khác. Với OAuth, người dùng có thể tinh chỉnh dữ liệu nào họ muốn chia sẻ thay vì phải giao toàn quyền kiểm soát tài khoản của họ cho bên thứ ba. Cơ chế này rất hay và tiện ích khi người dùng có thể sử dụng một tài khoản và đăng nhập trên nhiều nền tảng khác nhau.

OAuth và OAuth 2.0  là hai phiên bản khác nhau và OAuth 2.0 không được phát triển trên source của OAuth. Theo mình thấy thì nó khá hay ho và hữu ích. Tuy vậy, nó cũng tồn tại các "khe hở" khiến nó có thể bị attacker lợi dụng.

## OAuth 2.0

OAuth 2.0 được phát triển như một cách để chia sẻ kết nối với khối dữ liệu nhất định giữa các ứng dụng với nhau. Nó là trung gian giao tiếp giữa 3 bên bao gồm: chủ sở hữu data (data owner), ứng dụng khách (client application) và nhà cung cấp OAuth (OAuth provider).

### Cách OAuth 2.0 hoạt động
Để OAuth hoạt động thì có nhiều cách khác nhau và chúng thường được biết đến là 2 loại "flows" hoặc "grant types".

Tổng quan mà nói thì nó sẽ hoạt động chủ yếu theo 4 giai đoạn sau
1. Client Application yêu cầu kết nối đến một vùng dữ liệu nào đó của user. Trong yêu cầu đó thì cũng chỉ cụ thể là sẽ sử dụng loại OAuth nào và cách kết nối ra sao.
2. Người dùng đăng nhập vào dịch vụ OAuth sau đó đồng ý các yêu cầu của Client Application
3. Client Application nhận được một **unique access token** để nó chứng minh là nó có quyền kết nối đến data (user cho phép ở trên).
4. Client Application sử dụng token để khiến API nạp dữ liệu liên quan từ server.

### Một vài lỗ hổng với OAuth/OAuth2
Ở phía Client Application:
- Lỗ hổng triển khai không đúng cách xác nhận (grant type) ở client application
- Lỗ hổng CSRF

Ở phía OAuth service:
- Bị leak mã ủy quyền (authorization code) và mã truy cập (access token)
	- Xác thực redirect_uri không đúng luật
	- Bị đánh cắp mã và mã thông báo truy cập qua trang proxy
- Lỗ hổng xác thực phạm vị (scope validation)
- Không kiểm tra việc người dùng đăng ký

Ở phía user:
- Bị leak hoặc tấn công máy tính
- Đăng nhập vào các trang có mã độc
 
### Lab: Authentication bypass via OAuth implicit flow

- Giao diện trang web cho phép sử dụng mạng xã hội để đăng nhập:

  <img width="1919" height="943" alt="image" src="https://github.com/user-attachments/assets/63448710-444b-4990-9fc6-4ff666c776df" />

- Trong lab này, ta tiến hành đăng nhập với tài khoản social media có sẵn là: *wiener:peter*. Kết quả sau khi đăng nhập thành công:

  <img width="1919" height="880" alt="image" src="https://github.com/user-attachments/assets/5674761a-5561-4835-86a5-b2081efc2ed4" />

- Sau đó trang thực hiện redirect về trang chủ.

- Thực hiện kiểm tra Burp Suite để xem quy trình OAuth trên trang web. Và khi thực hiện nhập thành công tài khoản mạng xã hội ta thấy nó bắt đầu bằng một request xác thực *GET /auth?client_id=[...]* như sau:

  <img width="1431" height="516" alt="image" src="https://github.com/user-attachments/assets/157bdbc4-6c3e-4236-a179-b5c197278ad7" />

  Trong request trên, ta sẽ thấy:

  *client_id=ab8tzl43phra0tq2jlil* là	mã định danh của ứng dụng (client app) đã được đăng ký với OAuth provider.

  *redirect_uri=https://0a6f00d403a19fc80793a4005200e9.web-security-academy.net/oauth-callback*:	Địa chỉ callback mà Authorization Server sẽ redirect người dùng về sau khi xác thực xong.

  *response_type=token*: Đây là Implicit Flow (yêu cầu access token trực tiếp trong URL fragment thay vì code).

  *scope=openid&profile&email*:	Yêu cầu quyền truy cập thông tin OpenID, profile và email của user.

- Ta sẽ thấy OAuth Server sẽ lệnh redirect trình duyệt về redirect_uri mà client đã đăng ký (ở đây là /oauth-callback).

  <img width="1869" height="567" alt="image" src="https://github.com/user-attachments/assets/09a9a28a-bcae-4129-9f62-d8a61df498e6" />

  Fragment #access_token=... → là nơi token được trả về trực tiếp trên URL vì bạn đang dùng Implicit Flow (response_type=token ở request trước).

- Tiếp đó, trình duyệt gửi request để tải trang /oauth-callback từ server của ứng dụng.

  <img width="1873" height="604" alt="image" src="https://github.com/user-attachments/assets/07182004-9bbc-4bfc-9848-86e83c14ca53" />

  Có thể thấy, sử dụng token vừa lấy để gọi API /me → mục đích: lấy thông tin profile người dùng (OpenID, email, tên, …) từ nhà cung cấp OAuth.

  Sau khi lấy được thông tin user từ OAuth provider, script sẽ gửi POST request đến endpoint /authenticate của server ứng dụng.

- Cuối cùng, email và username được lấy từ API /me của OAuth provider sau khi xác thực thành công và token chính là access token mà ứng dụng đã lấy được từ OAuth server (bước trước):

  <img width="1866" height="606" alt="image" src="https://github.com/user-attachments/assets/491a16a5-edc4-4cd9-ae19-8338459af3da" />

  Sau đó redirect về trang chính /, tạo session mới cho user → bằng cách Set-Cookie.

- Tuy nhiên có thể thể thấy ở đây, web lại để client (trình duyệt) gửi thông tin user về qua endpoint /authenticate và server tin ngay giá trị thông tin user như email trong body request này là thật, không verify token với OAuth provider. Do đó nếu ta dùng Burp Repeater gửi lại request POST /authenticate, sau đó sửa giá trị "email" thành carlos@carlos-montoya.net và gửi lại request, lúc này server set cookie session cho Carlos. Sau đó, thực hiện chọn "Request in browser" > "In original session" để lấy URL này:

  <img width="1432" height="1001" alt="image" src="https://github.com/user-attachments/assets/6cf0462c-ba04-4d09-a742-45e39f80f630" />

  <img width="1919" height="885" alt="image" src="https://github.com/user-attachments/assets/1ca02820-3302-421e-9a0a-f6dda72e187f" />

  &rarr; Đăng nhập thành công tài khoản *carlos*

  

  








