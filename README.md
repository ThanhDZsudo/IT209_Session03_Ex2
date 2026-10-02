# Bài 2: Cấu hình trang lỗi tùy chỉnh (Custom Error Page 404) - IT209 DevOps

## 1. Mục tiêu bài tập
- Cấu hình dịch vụ Web Server Nginx phục vụ trang lỗi 404 tùy biến (`404.html`) thay cho giao diện mặc định.
- Thiết lập bảo mật bằng chỉ thị `internal` của Nginx để ngăn người dùng truy cập trực tiếp vào tệp `/404.html`.
- Nắm vững quy trình chuyển giao tệp (`scp`), phân quyền hệ thống (`chown`/`chmod`), và quản lý cấu hình máy chủ web (`sites-available` / `sites-enabled`).

---

## 2. Cấu trúc tệp tin trong bài nộp
Thư mục nộp bài trên GitHub: `homework/session_03/ex2/`
- [`404.html`](file:///e:/BaiTap/IT209_Devops/Session03/Ex2/404.html): Giao diện trang lỗi 404 tùy biến, thiết kế phong cách Cloud/DevOps responsive kèm khung chẩn đoán hệ thống.
- [`my-web.conf`](file:///e:/BaiTap/IT209_Devops/Session03/Ex2/my-web.conf): File cấu hình Server Block Nginx chứa các chỉ thị `error_page` và `internal`.
- [`README.md`](file:///e:/BaiTap/IT209_Devops/Session03/Ex2/README.md): Tài liệu hướng dẫn quy trình thực hiện và kết quả kiểm thử.

---

## 3. Quy trình thực hiện chi tiết

### Bước 1: Chuyển tệp tin từ máy tính cá nhân lên VPS (`scp`)
Trên cửa sổ Git Bash ở máy tính cá nhân (tại thư mục bài làm):
```bash
scp 404.html my-web.conf root@<IP_VPS>:~
```
> *Lệnh này sẽ chuyển an toàn 2 tệp tin vào thư mục Home (`/root`) của VPS.*

---

### Bước 2: Đưa tệp `404.html` vào Web Root và phân quyền
Truy cập vào VPS thông qua SSH và thực thi:
```bash
# 1. Tạo thư mục Web Root
sudo mkdir -p /var/www/my-web/html

# 2. Chuyển tệp 404.html vào thư mục Web Root
sudo mv ~/404.html /var/www/my-web/html/

# 3. Phân quyền sở hữu cho Nginx (người dùng www-data)
sudo chown -R www-data:www-data /var/www/my-web/html
sudo chmod 644 /var/www/my-web/html/404.html
```

---

### Bước 3: Cấu hình Nginx Server Block (`my-web.conf`)

Chuyển file cấu hình vào thư mục `sites-available`:
```bash
sudo mv ~/my-web.conf /etc/nginx/sites-available/my-web
```

Nội dung chi tiết trong `/etc/nginx/sites-available/my-web`:
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    # Thư mục web root chứa mã nguồn website và file 404.html
    root /var/www/my-web/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Chỉ thị cấu hình trang lỗi 404 tùy biến
    error_page 404 /404.html;

    # Bảo vệ file 404.html khỏi việc người dùng truy cập trực tiếp
    location = /404.html {
        root /var/www/my-web/html;
        internal;
    }
}
```

---

### Bước 4: Kích hoạt Site và xử lý xung đột cấu hình cũ
```bash
# Tạo symlink để kích hoạt site my-web
sudo ln -sf /etc/nginx/sites-available/my-web /etc/nginx/sites-enabled/

# Gỡ bỏ các site mặc định hoặc site của bài trước nếu đang cùng chiếm cổng 80
sudo rm -f /etc/nginx/sites-enabled/default
sudo rm -f /etc/nginx/sites-enabled/my-website
```

---

### Bước 5: Kiểm tra cú pháp và tải lại Nginx
```bash
# Kiểm tra tính toàn vẹn của cú pháp
sudo nginx -t

# Tải lại cấu hình Nginx mà không làm gián đoạn kết nối
sudo systemctl reload nginx
```
> Kết quả `nginx -t` thành công:
> ```text
> nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
> nginx: configuration file /etc/nginx/nginx.conf test is successful
> ```

---

## 4. Giải thích cơ chế kỹ thuật

1. **Chỉ thị `error_page 404 /404.html;`**:
   - Khi client yêu cầu một tài nguyên tĩnh không tồn tại trên máy chủ, Nginx phát hiện mã lỗi `404 Not Found` và tự động điều hướng xử lý nội bộ đến route `/404.html`.
2. **Chỉ thị `internal;` trong block `location = /404.html`**:
   - Chỉ thị `internal` quy định rằng route này **chỉ được phép gọi từ các tiến trình xử lý nội bộ của Nginx** (ví dụ do `error_page` chuyển đến).
   - Nếu client từ ngoài internet cố tình nhập trực tiếp URL `http://<IP_VPS>/404.html`, Nginx sẽ từ chối và trả về mã lỗi `404 Not Found`, bảo vệ cấu trúc và logic máy chủ.

---

## 5. Kết quả kiểm thử thực tế trên Droplet / Máy chủ VPS

### Kiểm tra 1: Truy cập một đường dẫn không tồn tại
```bash
curl -I http://localhost/invalid-path-demo
```
**Kết quả thực tế từ máy chủ:**
```http
HTTP/1.1 404 Not Found
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 07:51:53 GMT
Content-Type: text/html
Content-Length: 12570
Connection: keep-alive
ETag: "6abf6245-311a"
```
👉 *Trạng thái trả về đúng mã `404 Not Found`, kích thước `Content-Length: 12570` tương ứng đúng với dung lượng tệp `404.html` tùy chỉnh.*

---

### Kiểm tra 2: Truy cập trực tiếp vào tệp `/404.html`
```bash
curl -I http://localhost/404.html
```
**Kết quả thực tế từ máy chủ:**
```http
HTTP/1.1 404 Not Found
Server: nginx/1.18.0 (Ubuntu)
Date: Fri, 02 Oct 2026 07:51:53 GMT
Content-Type: text/html
Content-Length: 12570
Connection: keep-alive
ETag: "6abf6245-311a"
```
👉 *Chỉ thị `internal` hoạt động chính xác: Ngăn chặn người dùng xem trực tiếp với mã 200, mà ép trả về lỗi `404 Not Found`.*
