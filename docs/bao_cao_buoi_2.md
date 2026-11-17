# BÁO CÁO KẾT QUẢ THỰC HIỆN BÀI TẬP BUỔI 2
## Chủ đề: API, Chặn và Giả Lập Mạng, Chạy Song Song

---

## 1. TỔNG QUAN KẾT QUẢ

- **Tổng số test ban đầu**: 2 test passed, 14 test failed (`NotImplementedError`), 4 test deselected.
- **Tổng số test sau khi hoàn thiện**: **23 test passed (100%)**.
- **Tiêu chuẩn kiểm tra mã nguồn (Code standards)**:
  - `grep -rn "import requests" tests`: Không sử dụng thư viện `requests`, 100% sử dụng `APIRequestContext` của Playwright.
  - `grep -rn "https://" tests/buoi2`: Không có URL tuyệt đối hardcode trong logic mã kiểm thử (chỉ có trong comment/docstring).
  - Đã thêm từ khóa `"saw"` và `"wrench"` vào file `data/search_keywords.json`.
  - Đã kiểm tra chạy song song `-n 4` với `pytest-xdist`: hoàn thành 19 test trong ~9 giây.

---

## 2. CHI TIẾT CÁC BÀI TẬP ĐÃ HOÀN THÀNH

### Phần 1: Test API Toolshop (`tests/buoi2/test_b2_api.py`)

1. **B2.1 - Smoke test trạng thái API (`test_b2_1_status_returns_toolshop`)**:
   - Gửi `GET /status`.
   - Xác nhận HTTP Status = 200 và phản hồi JSON có `"app_name": "Toolshop"`.

2. **B2.2 - Kiểm tra hợp đồng JSON danh sách sản phẩm (`test_b2_2_products_first_page_has_pagination_and_product_shape`)**:
   - Gửi `GET /products?page=1`.
   - Kiểm tra cấu trúc hợp đồng (Contract Testing): các khóa phân trang (`current_page`, `data`, `per_page`, `total`, `last_page`).
   - Kiểm tra `0 < len(body["data"]) <= body["per_page"]`.
   - Kiểm tra từng sản phẩm: `id` là chuỗi hoặc số nguyên (`(str, int)` để tương thích cả PROD và BUGGY), `name` là chuỗi không rỗng, `price > 0`.
   - **Tuyệt đối không hardcode con số cụ thể** (như `total == 50`) để test chạy được trên mọi môi trường.

3. **B2.3 - Data-driven testing từ file JSON (`test_b2_3_search_api_returns_only_matching_names`)**:
   - Sử dụng `@pytest.mark.parametrize` đọc dữ liệu từ `data/search_keywords.json`.
   - Đã mở rộng file JSON với 4 trường hợp: `pliers`, `hammer`, `saw`, `wrench`.
   - Gửi `GET /products/search?q={keyword}`, kiểm tra danh sách kết quả không rỗng và tất cả tên sản phẩm trả về đều chứa từ khóa tìm kiếm.

4. **B2.4 - Kiểm thử các trường hợp lỗi 4xx**:
   - **B2.4a (`test_b2_4a_protected_or_missing_resource_returns_4xx`)**: Sử dụng `@pytest.mark.parametrize` kiểm tra endpoint cần xác thực `GET /users/me` trả về 401 Unauthorized và endpoint không tồn tại `GET /products/does-not-exist` trả về 404 Not Found.
   - **B2.4b (`test_b2_4b_register_empty_body_returns_422_with_field_errors`)**: Gửi `POST /users/register` với body rỗng `{}`, kiểm tra server trả mã 422 Unprocessable Entity kèm thông báo lỗi cho các trường `"email"` và `"password"`.
   - **B2.4c (`test_b2_4c_register_leaked_password_is_rejected`)**: Gửi đăng ký với mật khẩu phổ biến từng bị rò rỉ dữ liệu (`Welcome01!`), kiểm tra hệ thống từ chối với mã 422 và báo lỗi trường `password`.

5. **B2.5 - Quản lý Token & Bảo mật thông tin người dùng**:
   - **B2.5a (`test_b2_5a_users_me_with_token_returns_own_profile`)**: Gọi `GET /users/me` kèm header `Authorization: Bearer <token>` từ fixture `auth_headers`, xác nhận trả về thông tin profile có đúng email của `test_user`.
   - **B2.5b (`test_b2_5b_register_response_does_not_expose_password`)**: Đăng ký người dùng mới với mật khẩu mạnh, kiểm tra phản hồi trả về mã 201 Created và **không bao giờ để lộ trường `password`** trong body JSON.

6. **B2.6 - Custom Fixture tạo giỏ hàng (`new_cart`)**:
   - Viết fixture `new_cart(api)` phạm vi `function`: gửi `POST /carts`, xác nhận mã 201 và trả về `id` giỏ hàng mới.
   - Hàm test thêm sản phẩm vào giỏ hàng (`POST /carts/{id}` với quantity = 2), sau đó kiểm tra lại bằng `GET /carts/{id}` xem giỏ có đúng 1 sản phẩm với số lượng bằng 2.

---

### Phần 2: Can thiệp mạng và phối hợp API - UI (`tests/buoi2/test_b2_network.py`)

1. **B2.7 - Chặn tài nguyên ảnh (`test_b2_7_home_without_images_still_lists_products`)**:
   - Dùng `page.route()` với regex các định dạng ảnh `r"\.(png|jpe?g|webp|avif|svg)(\?.*)?$"` để gọi `route.abort()`.
   - Đăng ký route **trước khi gọi `open()`**.
   - Lưu lại danh sách các URL đã bị chặn và khẳng định `assert blocked` để chứng minh route thực sự đã kích hoạt.
   - Xác nhận danh sách tên sản phẩm vẫn hiển thị bình thường.

2. **B2.8 - Giả lập trạng thái rỗng (`test_b2_8_home_with_empty_product_list_shows_no_results`)**:
   - Dùng `page.route(PRODUCT_LIST, ...)` trả về `route.fulfill(status=200, body=json.dumps(EMPTY_PAGE))` để mock API trả về danh sách rỗng.
   - Kiểm tra UI hiển thị thông báo `"There are no products found."` và số lượng thẻ sản phẩm hiển thị là 0.

3. **B2.9 - Kiểm tra tính bền bỉ khi API gặp sự cố (`test_b2_9_home_when_product_api_fails_page_stays_usable`)**:
   - Dùng `page.route(PRODUCT_LIST, lambda route: route.fulfill(status=500))` mô phỏng lỗi máy chủ nội bộ 500.
   - Xác nhận trang không bị sập hoàn toàn: không có thẻ sản phẩm nào hiển thị, nhưng ô tìm kiếm (`search_box`) vẫn có thể nhập liệu (`to_be_editable()`).

4. **B2.10 - Sửa phản hồi mạng thật (`test_b2_10_home_renders_price_from_api_response`)**:
   - Lắng nghe request danh sách sản phẩm, gọi `route.fetch()` để lấy response từ server thật, sửa giá sản phẩm đầu tiên thành `999.99` rồi trả về bằng `route.fulfill(response=response, json=body)`.
   - Xác nhận giao diện hiển thị đúng giá tiền được sửa: `"$999.99"`.

5. **B2.11 - Tiêm Token bỏ qua Form đăng nhập (`test_b2_11_account_page_with_token_injected_skips_login_form`)**:
   - Sử dụng `page.add_init_script(...)` để ghi `auth-token` vào `window.localStorage` trước khi trang khởi động.
   - Mở thẳng trang `/account`, xác nhận người dùng đã ở trạng thái đăng nhập với tiêu đề `"My account"` và menu người dùng hiển thị `"QA Bot"`.

6. **B2.12 - Tái sử dụng trạng thái đăng nhập qua `storage_state`**:
   - Sử dụng fixture `logged_in_page` tái sử dụng phiên đã lưu từ file `storage_state` giúp tiết kiệm thời gian đăng nhập trên giao diện.

---

### Phần 3: Kiểm thử GitHub REST API (`tests/buoi2/test_b2_github_api.py`)

1. **B2.13a - Metadata của Public Repository (`test_b2_13a_github_repo_returns_expected_metadata`)**:
   - Gửi `GET /repos/microsoft/playwright-python`.
   - Xác nhận full_name, language (`Python`), license spdx_id (`Apache-2.0`), `stargazers_count > 0` và `archived is False`.

2. **B2.13b - Xử lý tài nguyên không tồn tại (`test_b2_13b_github_unknown_user_returns_404_with_message`)**:
   - Dùng `uuid.uuid4().hex` sinh tên tài khoản không thể tồn tại.
   - Gửi `GET /users/{unknown_user}`, xác nhận status 404 và body JSON chứa `"message": "Not Found"`.

3. **B2.13c - Đọc Rate-Limit Headers (`test_b2_13c_github_response_reports_rate_limit`)**:
   - Gửi `GET /rate_limit` (endpoint không bị trừ hạn ngạch).
   - Đọc các headers viết thường: `x-ratelimit-limit` và `x-ratelimit-remaining`, ép kiểu sang số nguyên và kiểm tra `remaining <= limit`.

4. **B2.13d - So khớp phiên bản Release với thư viện Playwright cục bộ (`test_b2_13d_github_latest_release_matches_installed_playwright`)**:
   - Đọc phiên bản Playwright đang cài đặt trong môi trường ảo thông qua `importlib.metadata.version("playwright")`.
   - Gọi `GET /repos/microsoft/playwright-python/releases/latest`, kiểm tra tag_name có dạng `vX.Y.Z` và phiên bản release mới nhất trên GitHub không cũ hơn phiên bản đang cài đặt cục bộ bằng `packaging.version.Version`.

---

## 3. GIẢI ĐÁP CHI TIẾT 8 CÂU HỎI TỰ KIỂM TRA (MỤC 12)

### Câu 1: `assert response.ok` và `assert response.status == 201` khác nhau thế nào? Khi nào dùng cái nào?
- **Sự khác nhau**:
  - `response.ok` trả về `True` cho mọi mã HTTP trong dải thành công `2xx` (200 OK, 201 Created, 204 No Content...).
  - `response.status == 201` kiểm tra chính xác mã trả về phải là `201 Created`.
- **Khi nào dùng**:
  - Dùng `response.status == 201` khi kiểm tra các API tạo mới tài nguyên (như `POST /users/register`, `POST /carts`). Việc kiểm tra mã cụ thể giúp phát hiện lỗi logic của backend (ví dụ tạo mới nhưng server lại trả nhầm 200 thay vì 201). Nguyên tắc: *"Assertion càng cụ thể càng bắt được nhiều lỗi"*.
  - Dùng `response.ok` khi chỉ cần xác nhận thao tác thành công chung chung mà không cần ràng buộc mã cụ thể.

### Câu 2: Vì sao test API của nhóm không nên kiểm `total == 50`?
- Hệ thống Toolshop có nhiều môi trường khác nhau: môi trường PROD có 50 sản phẩm ban đầu, nhưng BUGGY chỉ có 26 sản phẩm. Hơn nữa, đây là môi trường test dùng chung, dữ liệu có thể biến động theo thời gian.
- Nếu kiểm tra con số tuyệt đối `total == 50`, test sẽ bị vỡ (false failure) khi chuyển đổi môi trường hoặc khi dữ liệu thay đổi. Vì vậy, ta cần thực hiện **Contract Testing** (kiểm thử hợp đồng): kiểm tra cấu trúc schema, sự tồn tại của các key, kiểu dữ liệu (`id` là `str` hoặc `int`, `price > 0`, `0 < len(data) <= per_page`) thay vì kiểm tra con số cố định.

### Câu 3: Chạy `pytest -n 4`, fixture `test_user` chạy mấy lần? Vì sao email phải ngẫu nhiên?
- Khi chạy `pytest -n 4` với `pytest-xdist`, mỗi worker là một tiến trình Python độc lập với phiên pytest riêng. Do đó fixture có `scope="session"` sẽ được chạy **1 lần trên mỗi worker**, tức là chạy **4 lần** (tạo 4 tài khoản khác nhau).
- Email phải được tạo ngẫu nhiên (bằng `new_email()`) để tránh việc các worker cố gắng đăng ký cùng một địa chỉ email tại cùng một thời điểm, dẫn đến lỗi xung đột dữ liệu `409 Conflict: "A customer with this email address already exists."`.

### Câu 4: Vì sao route phải đăng ký trước khi mở trang?
- Trong Playwright, `page.route()` thiết lập cơ chế chặn bắt (intercept) ở tầng mạng của trình duyệt. Nếu gọi `page.goto()` hoặc `open()` trước rồi mới đăng ký route, các network request ban đầu (như tải HTML, CSS, ảnh, gọi API ban đầu) đã được gửi đi và nhận kết quả xong từ trước đó. Kết quả là route được đăng ký sau sẽ không có tác dụng gì đối với các request đã xảy ra.

### Câu 5: Glob `**/products?*` có bắt được request tải danh sách sản phẩm trên PROD không? Vì sao?
- **Không bắt được**.
- Vì trên môi trường PROD, trang chủ tải danh sách sản phẩm bằng HTTP method `QUERY` tới endpoint `https://api.practicesoftwaretesting.com/products` (tham số lọc được đặt trong body request, URL hoàn toàn không có dấu `?` hay query string). Glob `**/products?*` bắt buộc URL phải có ký tự `?`, do đó nó sẽ bỏ sót request trên PROD. Cần dùng biểu thức chính quy (regex) `re.compile(r"/products(\?.*)?$")` để khớp cả hai trường hợp có hoặc không có query string.

### Câu 6: `storage_state` tiết kiệm được gì, và có rủi ro gì?
- **Tiết kiệm**: Tiết kiệm đáng kể thời gian chạy test UI. Thay vì mỗi bài test đều phải mở form đăng nhập, điền email, mật khẩu rồi bấm submit, fixture `auth_state` đăng nhập qua UI một lần duy nhất trong toàn bộ phiên test và lưu cookie + localStorage ra file. Các test sau chỉ cần nạp lại file trạng thái này là có sẵn phiên đăng nhập.
- **Rủi ro**:
  - Token hoặc session lưu trong file có thể bị hết hạn (expired) giữa chừng nếu bộ test chạy quá lâu.
  - File `storage_state.json` chứa thông tin nhạy cảm (JWT token, session cookie), nếu vô tình commit lên Git repository sẽ gây nguy cơ lộ bảo mật.

### Câu 7: Kể hai tình huống nên kiểm thử qua API thay vì qua giao diện.
- **Tình huống 1**: Kiểm thử các quy tắc nghiệp vụ (business logic), phân quyền truy cập, mã lỗi HTTP (401, 403, 404, 422) và xác thực dữ liệu từng trường (field validation). Kiểm thử qua API không phụ thuộc vào giao diện hiển thị, chạy nhanh hơn và không bị vỡ vặt khi UI thay đổi thiết kế/HTML.
- **Tình huống 2**: Chuẩn bị dữ liệu ban đầu (test setup / precondition) cho test UI. Ví dụ: tạo tài khoản mới, tạo giỏ hàng, nạp tiền, đặt đơn hàng mẫu. Thay vì tốn hàng chục bước click trên UI, gọi API trực tiếp giúp tạo trạng thái chỉ trong vài mili-giây.

### Câu 8: Vì sao test GitHub không kiểm `stargazers_count == 15084`? Test đỏ vì "rate limit exceeded" là lỗi của ai?
- Không kiểm `stargazers_count == 15084` vì số sao của kho lưu trữ công khai trên GitHub biến động liên tục theo thời gian (người dùng bấm star/unstar). Do đó chỉ kiểm tra kiểu dữ liệu và giá trị hợp lý: `isinstance(count, int) and count > 0`.
- Test đỏ vì *"rate limit exceeded"* (mã lỗi 403 hoặc 429) **không phải là lỗi của mã kiểm thử (test bug) hay lỗi của ứng dụng**, mà là do giới hạn hạn ngạch gọi API của dịch vụ bên ngoài (GitHub giới hạn 60 lượt gọi/giờ cho IP không có token). Để khắc phục, cần thêm `GITHUB_TOKEN` vào file `.env`.

---

## 4. HƯỚNG DẪN LỆNH CHẠY KIỂM TRA

Mở terminal trong thư mục dự án và chạy các lệnh sau bằng môi trường ảo `.venv`:

```bash
# 1. Chạy toàn bộ 23 bài test buổi 2 với thông tin chi tiết:
.venv/bin/pytest tests/buoi2 -v

# 2. Chạy riêng từng file:
.venv/bin/pytest tests/buoi2/test_b2_api.py -v
.venv/bin/pytest tests/buoi2/test_b2_network.py -v
.venv/bin/pytest tests/buoi2/test_b2_github_api.py -v

# 3. Chạy xem giao diện trình duyệt thực tế (headed mode):
.venv/bin/pytest tests/buoi2/test_b2_network.py -k b2_8 --headed --slowmo 300

# 4. Chạy song song 4 tiến trình (pytest-xdist):
.venv/bin/pytest tests/buoi2 -m "not github" -n 4

# 5. Kiểm tra tính chuẩn mực mã nguồn:
grep -rn "import requests" tests      # Phải không có dòng nào in ra
grep -rn "https://" tests/buoi2       # Đúng 3 dòng comment/docstring
```
