# Báo Cáo Bài Tập 3: Bảo Mật Tài Nguyên Bằng HTTP Basic Authentication

- **Môn học:** IT209
- **Buổi học:** Session 03 - Server Security & Hardening
- **Bài tập:** Bài 3 - Bảo mật tài nguyên bằng HTTP Basic Authentication
- **Học viên:** Hoang Thien Son (hoangthienson06@gmail.com)

---

## 1. Mục tiêu bài thực hành
- **Bảo vệ tài nguyên nhạy cảm:** Thiết lập lớp xác thực đăng nhập người dùng cho thư mục quản trị nội bộ (`/admin`) nhằm ngăn chặn người dùng từ Internet truy cập trái phép.
- **Áp dụng cơ chế HTTP Basic Authentication:** Cấu hình Web Server Nginx yêu cầu thông tin định danh (Username/Password) trực tiếp thông qua hộp thoại chuẩn của trình duyệt.
- **Bảo mật lưu trữ mật khẩu:** Sử dụng thuật toán băm mật khẩu chuẩn APR1/MD5 qua công cụ `htpasswd`, lưu trữ tệp `.htpasswd` tại thư mục cấu hình hệ thống `/etc/nginx/`, tách biệt hoàn toàn khỏi thư mục mã nguồn web `/var/www/` để phòng tránh nguy cơ rò rỉ tệp chứa hash mật khẩu.

---

## 2. Thông số thiết lập dịch vụ

| Thành phần | Giá trị cấu hình | Ý nghĩa kỹ thuật |
| :--- | :--- | :--- |
| **Đường dẫn tài nguyên bảo vệ** | `/admin` | Thư mục trang quản trị nội bộ |
| **Tài khoản quản trị** | `admin_user` | Tài khoản được cấp quyền truy cập |
| **Tệp mật khẩu băm** | `/etc/nginx/.htpasswd` | Tách biệt khỏi Document Root |
| **Phân quyền tệp `.htpasswd`** | `640` (`chown www-data:www-data`) | Chỉ Nginx worker process được đọc |
| **Thông điệp xác thực (Realm)** | `"Restricted Admin Area"` | Hiển thị trên popup trình duyệt |
| **Mã trạng thái khi chưa xác thực** | `401 Unauthorized` | Chặn truy cập và yêu cầu đăng nhập |
| **Mã trạng thái khi xác thực đúng** | `200 OK` | Cho phép truy cập vào nội dung |

---

## 3. Các bước triển khai chi tiết

### Bước 1: Cài đặt công cụ tạo mã băm `apache2-utils`
Tiện ích `htpasswd` nằm trong gói `apache2-utils` trên hệ điều hành Ubuntu:
```bash
sudo apt update -y
sudo apt install apache2-utils -y
```

---

### Bước 2: Tạo tệp mật khẩu ẩn `.htpasswd` và tài khoản `admin_user`
Sử dụng cờ `-c` (create new file) để tạo mới file mật khẩu ẩn tại `/etc/nginx/`:
```bash
sudo htpasswd -b -c /etc/nginx/.htpasswd admin_user Boizi06@
```

Thiết lập phân quyền bảo mật: Chỉ cho phép người dùng hệ thống chạy Nginx (`www-data`) đọc file:
```bash
sudo chown www-data:www-data /etc/nginx/.htpasswd
sudo chmod 640 /etc/nginx/.htpasswd
```

Kiểm tra nội dung tệp mật khẩu đã được băm an toàn:
```bash
sudo cat /etc/nginx/.htpasswd
```
> **Output:**
> ```text
> admin_user:$apr1$R6RtNwGU$JXD1s2kUZqqYkHrwKSBB60
> ```

---

### Bước 3: Cấu hình Server Block Nginx (`my-web.conf`)
Chỉnh sửa file cấu hình tại `/etc/nginx/sites-available/my-web.conf`, bổ sung khối `location /admin`:
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/my-web/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Cấu hình bảo vệ khu vực quản trị nội bộ bằng HTTP Basic Authentication
    location /admin {
        auth_basic "Restricted Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        try_files $uri $uri/ =404;
    }

    # Cấu hình trang lỗi tùy chỉnh 404 (từ Bài 2)
    error_page 404 /404.html;
    location = /404.html {
        root /var/www/my-web/html;
        internal;
    }

    access_log /var/log/nginx/my-web.access.log;
    error_log /var/log/nginx/my-web.error.log;
}
```

---

### Bước 4: Khởi tạo thư mục quản trị và trang hiển thị
Tạo thư mục `/var/www/my-web/html/admin/` và file `index.html` dành cho quản trị viên:
```bash
sudo mkdir -p /var/www/my-web/html/admin
sudo chown -R www-data:www-data /var/www/my-web/html/admin
```

---

### Bước 5: Kiểm tra cú pháp và nạp lại cấu hình dịch vụ
```bash
sudo nginx -t
```
> **Output:**
> ```text
> nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
> nginx: configuration file /etc/nginx/nginx.conf test is successful
> ```

Nạp lại cấu hình:
```bash
sudo systemctl reload nginx
```

---

## 4. Kết quả kiểm tra & Terminal Logs

### 4.1. Kiểm tra yêu cầu khi không cung cấp thông tin xác thực:
Chạy lệnh curl kiểm tra header:
```bash
curl -I http://165.101.46.48/admin
```
**Nhật ký Terminal hiển thị:**
```text
HTTP/1.1 401 Unauthorized
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 01:11:36 GMT
Content-Type: text/html
Content-Length: 188
Connection: keep-alive
WWW-Authenticate: Basic realm="Restricted Admin Area"
```
👉 **Đánh giá:** Nginx từ chối truy cập với mã `401 Unauthorized` và trả về header `WWW-Authenticate`, kích hoạt hộp thoại đăng nhập của trình duyệt.

---

### 4.2. Kiểm tra yêu cầu kèm tài khoản mật khẩu hợp lệ:
Chạy lệnh curl kèm tùy chọn xác thực `-u admin_user:<password>`:
```bash
curl -I -u admin_user:Boizi06@ http://165.101.46.48/admin/
```
**Nhật ký Terminal hiển thị:**
```text
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 01:11:36 GMT
Content-Type: text/html
Content-Length: 4845
Last-Modified: Fri, 02 Oct 2026 01:11:15 GMT
Connection: keep-alive
ETag: "6abf04b3-12ed"
Accept-Ranges: bytes
```
👉 **Đánh giá:** Xác thực thành công và trả về mã `200 OK`, người dùng truy cập được trang quản trị nội bộ.

---

### 4.3. Kiểm tra thực tế trên trình duyệt Web:
1. Truy cập địa chỉ: 👉 **`http://165.101.46.48/admin`**
2. Trình duyệt hiển thị hộp thoại pop-up đăng nhập yêu cầu:
   - **Tên người dùng:** `admin_user`
   - **Mật khẩu:** `Boizi06@`
3. Sau khi nhấn "Đăng nhập", giao diện mở ra trang **PTIT Admin Dashboard** với trạng thái `Authorized (200 OK)`.

---

## 5. Danh mục tệp tin nộp bài
Các tệp đã được tạo và lưu trữ đầy đủ trong thư mục:
```text
Session_03/
└── ex3/
    ├── .htpasswd          # Tệp mật khẩu mẫu chứa mã hash admin_user
    ├── my-web.conf        # File cấu hình Server Block Nginx chứa auth_basic
    ├── admin_index.html   # Giao diện trang quản trị /admin
    └── README.md          # Báo cáo thực hiện chi tiết
```
