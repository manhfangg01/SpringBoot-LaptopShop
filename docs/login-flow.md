# Luồng xử lý Login

Tài liệu này mô tả luồng đăng nhập hiện tại của ứng dụng Laptopshop sau khi đã chuẩn hóa lại theo một đường xác thực duy nhất qua Spring Security.

## 1. Mở trang login

- Người dùng truy cập `GET /login`.
- `HomePageController#getLoginPage()` trả về view `client/auth/login`.
- Form login có các trường:
  - `username` cho email
  - `password` cho mật khẩu
  - `remember-me` cho tùy chọn ghi nhớ đăng nhập
- Form submit trực tiếp về `POST /login`.

## 2. Spring Security xử lý request

Request `POST /login` không đi qua controller riêng. Spring Security sẽ bắt request này và xử lý trong `SecurityConfiguration`.

Cấu hình chính:

- `loginPage("/login")`
- `loginProcessingUrl("/login")`
- `usernameParameter("username")`
- `passwordParameter("password")`
- `failureUrl("/login?error")`
- `successHandler(customSuccessHandler())`

`DaoAuthenticationProvider` được cấu hình với:

- `CustomUserDetailsService`
- `BCryptPasswordEncoder`

## 3. Load user và xác thực

Luồng xác thực chuẩn hiện tại là:

1. Spring Security lấy giá trị từ field `username`.
2. Gọi `CustomUserDetailsService#loadUserByUsername(username)`.
3. Service tìm user theo email bằng `userService.getUserByEmail(username)`.
4. Nếu không tìm thấy user, ném `UsernameNotFoundException`.
5. Nếu tìm thấy user, Spring Security so khớp password bằng `BCryptPasswordEncoder`.
6. Nếu password đúng, request được coi là authenticated.

Phần validate login bằng `LoginDTO` / `LoginValidator` đã được loại bỏ để tránh trùng lặp và tránh hai cơ chế kiểm tra khác nhau cùng chạy cho một form login.

## 4. Đăng nhập thành công

Khi xác thực thành công, `CustomSuccessHandler` sẽ được gọi.

Handler này:

- Xóa thông tin lỗi authentication cũ trong session nếu có.
- Lấy lại user theo email từ DB.
- Ghi các thông tin cần dùng vào session:
  - `fullName`
  - `avatar`
  - `id`
  - `email`
  - `address`
  - `sum` của cart, nếu có

Sau đó hệ thống điều hướng theo role:

- `ROLE_USER` → `/`
- `ROLE_ADMIN` → `/admin`

## 5. Đăng nhập thất bại

Nếu xác thực thất bại, Spring Security redirect về:

- `/login?error`

Trên giao diện login, khi `param.error` tồn tại thì hiển thị:

- `Invalid email or password.`

Nếu logout thành công, trang login hiển thị:

- `Logout success`

## 6. Các rule bảo mật liên quan

- `/admin/**` chỉ cho phép `ROLE_ADMIN`
- `/register`, `/login`, `/client/**`, `/css/**`, `/js/**`, `/images/**`, `/product/**`, `/add-product-to-cart/**`, `/products/**` được public
- Các request còn lại yêu cầu đã authenticated

Session cũng được cấu hình để:

- luôn tạo session khi cần (`SessionCreationPolicy.ALWAYS`)
- giới hạn 1 session cho 1 account
- session hết hạn sẽ redirect tới `/logout?expired`

## 7. Ghi chú quan trọng

- Hiện tại checkbox `remember-me` đã được nối đúng tên param chuẩn của Spring Security.
- Luồng login chỉ còn một cơ chế xác thực chính: Spring Security + `CustomUserDetailsService` + `BCryptPasswordEncoder`.
- Nếu người dùng truy cập `/admin/**` khi chưa đăng nhập, hệ thống sẽ yêu cầu login trước khi cho đi tiếp.

## Tóm tắt ngắn

Luồng hiện tại là: mở `/login` → submit form → Spring Security gọi `CustomUserDetailsService` để load user → xác thực bằng password hash → thành công thì `CustomSuccessHandler` set session và redirect theo role, thất bại thì quay lại `/login?error`.
