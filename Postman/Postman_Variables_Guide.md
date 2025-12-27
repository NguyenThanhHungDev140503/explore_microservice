# Hướng dẫn Chi tiết: Làm việc với Biến (Variables) trong Postman

Tài liệu này cung cấp hướng dẫn toàn diện về cách sử dụng Biến trong Postman, được tổng hợp và biên soạn dựa trên tài liệu chính thức của Postman.

## 1. Giới thiệu về Biến trong Postman

**Biến (Variable)** trong Postman cho phép bạn lưu trữ và tái sử dụng các giá trị ở nhiều nơi khác nhau. Thay vì phải nhập đi nhập lại một giá trị (như base URL, API Key, hoặc ID của user) một cách thủ công, bạn có thể lưu nó vào một biến và tham chiếu tên biến đó trong các Request, Script, hoặc Environment.

**Lợi ích chính:**
*   **Hiệu quả & Tiết kiệm thời gian:** Thay đổi giá trị ở một nơi, cập nhật cho toàn bộ dự án.
*   **Linh hoạt:** Dễ dàng chuyển đổi giữa các môi trường (Ví dụ: từ Local -> Staging -> Production) mà không cần sửa đổi Request.
*   **Dễ đọc:** Request URL gọn gàng và dễ hiểu hơn (Ví dụ: `{{base_url}}/users` thay vì `https://api.staging.example.com/v1/users`).

---

## 2. Phạm vi của Biến (Variable Scopes)

Postman hỗ trợ 5 loại phạm vi biến khác nhau, được xếp hạng theo **độ ưu tiên** từ thấp đến cao. Khi có nhiều biến trùng tên, biến ở phạm vi hẹp hơn (độ ưu tiên cao hơn) sẽ được sử dụng.

### Các loại phạm vi (Từ rộng đến hẹp)

1.  **Global (Toàn cục):**
    *   Phạm vi rộng nhất.
    *   Có thể truy cập từ bất kỳ Collection, Request, hoặc Script nào trong Workspace.
    *   *Dùng cho:* Các hằng số chung không thay đổi, hoặc dữ liệu dùng chung cho mọi dự án.

2.  **Collection (Bộ sưu tập):**
    *   Chỉ có hiệu lực trong Collection chứa nó.
    *   Không phụ thuộc vào Environment đang chọn.
    *   *Dùng cho:* Các giá trị đặc thù của một API/Dự án cụ thể (Ví dụ: đường dẫn API prefix chung của dự án đó).

3.  **Environment (Môi trường):**
    *   Liên kết với một môi trường cụ thể (ví dụ: Dev, Test, Prod).
    *   Chỉ truy cập được khi bạn **chọn** Environment đó.
    *   *Dùng cho:* Cấu hình thay đổi theo môi trường như `base_url`, `username`, `password`, `api_token`.

4.  **Data (Dữ liệu):**
    *   Chỉ có sẵn khi chạy **Collection Runner**.
    *   Lấy từ file dữ liệu bên ngoài (JSON hoặc CSV) được nạp vào khi chạy test.
    *   *Dùng cho:* Testing với nhiều bộ dữ liệu đầu vào khác nhau (Data-driven testing).

5.  **Local (Cục bộ):**
    *   Chỉ tồn tại trong thời gian chạy của script (Pre-request Script hoặc Tests).
    *   Mất đi sau khi script chạy xong.
    *   *Dùng cho:* Ghi đè tạm thời các giá trị biến khác mà không muốn lưu lại lâu dài.

### Sơ đồ độ ưu tiên (Precedence)

Nếu bạn có một biến tên `username` ở cả Global và Environment, Postman sẽ lấy giá trị ở **Environment**.

`Local` > `Data` > `Environment` > `Collection` > `Global`

---

## 3. Các loại Biến (Variable Types)

Khi định nghĩa biến (đặc biệt trong Environment và Global), bạn có thể chọn loại:

*   **Default:** Giá trị được lưu dưới dạng văn bản thuần (plain text).
*   **Secret:** Giá trị sẽ được ẩn đi (masked) trên giao diện (hiện các dấu `***`). Dùng để lưu mật khẩu, khóa bí mật. Postman sẽ cảnh báo nếu bạn cố gắng 'Persist' (lưu cứng) các biến Secret này lên Cloud để chia sẻ với team, giúp bảo mật hơn.

---

## 4. Cách sử dụng Biến

### 4.1. Trong Request Builder
Bạn có thể sử dụng biến ở bất kỳ đâu trong trình soạn thảo Request (URL, Params, Headers, Body, Authorization...) bằng cú pháp hai dấu ngoặc nhọn:

`{{variable_name}}`

**Ví dụ:**
*   URL: `{{base_url}}/api/v1/users`
*   Header: `Authorization`: `Bearer {{access_token}}`
*   Body (JSON):
    ```json
    {
      "name": "User {{user_id}}",
      "role": "{{user_role}}"
    }
    ```

Khi di chuột qua `{{variable_name}}`, Postman sẽ hiển thị giá trị hiện tại của biến đó và phạm vi của nó. Nếu biến chưa được định nghĩa, nó sẽ hiện màu đỏ.

### 4.2. Trong Scripts (Pre-request & Tests)
Trong các tab script, bạn dùng đối tượng `pm.*` để thao tác với biến.

*   **Truy cập biến (theo độ ưu tiên):**
    ```javascript
    // Tự động tìm biến theo thứ tự ưu tiên (Local > Envir > ... > Global)
    let myValue = pm.variables.get("variable_name");
    ```

*   **Truy cập biến theo phạm vi cụ thể:**
    ```javascript
    let glob = pm.globals.get("variable_name");
    let env = pm.environment.get("variable_name");
    let coll = pm.collectionVariables.get("variable_name");
    ```

*   **Thiết lập giá trị biến (Setters):**
    ```javascript
    // Set biến Global
    pm.globals.set("variable_name", "value");

    // Set biến Collection
    pm.collectionVariables.set("variable_name", "value");

    // Set biến Environment
    pm.environment.set("variable_name", "value");

    // Set biến Local (chỉ tồn tại trong lần chạy này)
    pm.variables.set("variable_name", "value");
    ```

---

## 5. Biến Động (Dynamic Variables)

Postman cung cấp sẵn các biến động để giúp bạn tạo dữ liệu mẫu hoặc dữ liệu ngẫu nhiên (random) cho các request. Bạn không cần định nghĩa chúng, chỉ cần gọi ra dùng.

Cú pháp: `{{$variable_name}}` (có dấu `$` ở trước).

**Một số biến động phổ biến:**

*   **ID & Số:**
    *   `{{$guid}}`: Tạo một UUID v4 ngẫu nhiên (Ví dụ: `d94a9762-5883-4a16...`).
    *   `{{$randomInt}}`: Số nguyên ngẫu nhiên từ 0 đến 1000.
    *   `{{$timestamp}}`: Timestamp hiện tại (Unix timestamp).

*   **Internet & Web:**
    *   `{{$randomEmail}}`: Tạo email ngẫu nhiên.
    *   `{{$randomUrl}}`: Tạo URL ngẫu nhiên.
    *   `{{$randomIP}}`: Tạo địa chỉ IP ngẫu nhiên.

*   **Tên & Thông tin người:**
    *   `{{$randomFirstName}}`: Tên (First name).
    *   `{{$randomLastName}}`: Họ (Last name).
    *   `{{$randomFullName}}`: Họ và tên đầy đủ.

*   **Màu sắc & Khác:**
    *   `{{$randomColor}}`: Tên màu (ví dụ: "red").
    *   `{{$randomHexColor}}`: Mã màu Hex (ví dụ: "#a4b5c6").

*Cách dùng trong Script:* Hiện tại trong script, bạn không thể gọi trực tiếp `pm.variables.get("$guid")` mà nên dùng thư viện `pm.variables.replaceIn()` để phân giải chuỗi chứa biến động:
```javascript
let randomEmail = pm.variables.replaceIn("{{$randomEmail}}");
console.log(randomEmail);
```

---

## 6. Làm việc với Environments (Môi trường)

**Environment** là một nhóm các biến được thiết lập cho một mục đích làm việc cụ thể. Đây là cách tốt nhất để quản lý sự khác biệt giữa các môi trường triển khai (Deployment environments).

### Tạo và Quản lý Environment
1.  Bấm vào mục **Environments** ở thanh sidebar bên trái.
2.  Bấm **+** để tạo mới.
3.  Đặt tên (ví dụ: "Staging", "Production").
4.  Thêm các biến (`base_url`, `api_key`...) và giá trị tương ứng.
    *   **Initial Value:** Giá trị này được đồng bộ lên server Postman (nếu đăng nhập). Dùng để chia sẻ với team.
    *   **Current Value:** Giá trị cục bộ trên máy bạn. Postman SỬ DỤNG giá trị này khi gửi request. Nó **KHÔNG** được đồng bộ lên server (trừ khi bạn chọn Persist All).

### Sử dụng Environment
*   Ở góc trên bên phải giao diện Postman, có một menu thả xuống (Dropdown) để chọn Environment đang hoạt động (`No Environment` hoặc tên Environment bạn đã tạo).
*   Khi chọn một Environment, postman sẽ load các biến trong Environment đó để sử dụng.

---

## 7. Thực hành tốt nhất (Best Practices)

1.  **Luôn sử dụng Environment cho URL:** Đừng bao giờ hardcode `http://localhost:3000` hay `https://api.mydomain.com` trực tiếp trong Request. Hãy luôn dùng `{{base_url}}` và tạo các Environment tương ứng.
2.  **Bảo mật:** Đánh dấu các biến chứa token, password là **Secret**. Hạn chế lưu các thông tin nhạy cảm vào `Initial Value` nếu bạn làm việc trong team public hoặc chia sẻ collection ra ngoài.
3.  **Đặt tên biến nhất quán:** Sử dụng quy tắc đặt tên rõ ràng (ví dụ: `snake_case` như `user_id`, `access_token`) để dễ đọc và tránh nhầm lẫn.
4.  **Clean up:** Xóa các biến Global hoặc Environment không còn sử dụng để tránh "rác" và xung đột không đáng có.

---
*Tài liệu này được biên soạn cho mục đích tham khảo nội bộ và học tập.*
