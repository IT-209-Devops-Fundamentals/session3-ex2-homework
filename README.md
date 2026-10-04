# BÀI TẬP 2: CẤU HÌNH TRANG LỖI TÙY CHỈNH (CUSTOM ERROR PAGE 404)

**Mã bài tập:** SESSION 03 - EXERCISE 2 (ss03-ex2)  
**Môn học:** IT215 - Cloud Infrastructure & DevOps Fundamentals  
**Web Server:** Nginx trên Ubuntu Server 24.04 LTS  
**Đường dẫn Web Root:** `/var/www/my-web/html/`  
**Cơ chế bảo mật:** Chỉ thị `internal` của Nginx chặn truy cập trực tiếp `/404.html`  

---

## 1. MỤC TIÊU BÀI TẬP

1. **Nâng cao Trải nghiệm Người dùng (UX Enhancement):** Thay thế trang lỗi mặc định đơn điệu, thô sơ của Nginx (*"404 Not Found - nginx/1.24..."*) bằng một giao diện HTML tùy biến chuyên nghiệp, thân thiện.
2. **Cấu hình Server Block Nginx với `error_page`:** Thiết lập ánh xạ mã lỗi HTTP `404 Not Found` sang tệp `/404.html`.
3. **Bảo mật với chỉ thị `internal`:** Đảm bảo tệp `/404.html` chỉ được gọi ngầm nội bộ bởi Nginx khi phát sinh lỗi 404 thực sự, ngăn chặn người dùng ngoài Internet truy cập trực tiếp bằng đường dẫn `http://<IP>/404.html`.

---

## 2. BẢNG TỔNG HỢP CẤU HÌNH & KẾT QUẢ MONG ĐỢI

| Yêu cầu kiểm tra | Lệnh thực hiện | HTTP Status Code | Kết quả giao diện / Nội dung |
| :--- | :--- | :--- | :--- |
| **Truy cập đường dẫn không tồn tại** | `curl -I http://<IP>/invalid-path-demo` | **`HTTP/1.1 404 Not Found`** | Hiển thị nội dung tùy biến từ `404.html` |
| **Truy cập trực tiếp file lỗi** | `curl -I http://<IP>/404.html` | **`HTTP/1.1 404 Not Found`** | Nginx chặn truy cập trực tiếp nhờ `internal` |

---

## 3. MÔ HÌNH HOẠT ĐỘNG CỦA CHỈ THỊ `internal` TRONG NGINX

```mermaid
sequenceDiagram
    autonumber
    actor Client as Người dùng (Trình duyệt / cURL)
    participant Nginx as Nginx Web Server
    participant FS as File System (/var/www/my-web/html/)

    rect rgb(255, 240, 240)
        Note over Client, FS: Kịch bản 1: Truy cập đường dẫn sai (/invalid-path-demo)
        Client->>Nginx: GET /invalid-path-demo
        Nginx->>FS: Tìm file /invalid-path-demo
        FS-->>Nginx: Không tìm thấy (File not found)
        Nginx->>Nginx: Kích hoạt chỉ thị error_page 404 /404.html (Chuyển tiếp nội bộ)
        Nginx->>FS: Đọc nội dung file /var/www/my-web/html/404.html
        FS-->>Nginx: Trả về nội dung 404.html
        Nginx-->>Client: HTTP/1.1 404 Not Found + Giao diện 404 tùy biến
    end

    rect rgb(240, 245, 255)
        Note over Client, FS: Kịch bản 2: Cố tình truy cập trực tiếp (/404.html)
        Client->>Nginx: GET /404.html
        Nginx->>Nginx: Khớp location = /404.html có chỉ thị "internal;"
        Note over Nginx: Chỉ thị internal TỪ CHỐI mọi request trực tiếp từ client ngoài!
        Nginx-->>Client: HTTP/1.1 404 Not Found (Chặn an toàn)
    end
```

---

## 4. CHI TIẾT CÁC BƯỚC THỰC HIỆN TRÊN MÁY CHỦ

### Bước 1: Chuẩn bị thư mục Web Root và tệp tin `404.html`

Tạo thư mục web root `/var/www/my-web/html/` (nếu chưa có):
```bash
sudo mkdir -p /var/www/my-web/html
```

Tạo tệp tin `/var/www/my-web/html/404.html`:
```bash
sudo nano /var/www/my-web/html/404.html
```

Dán nội dung HTML tùy biến vào file:
```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Không tìm thấy trang | PTIT DevOps</title>
    <style>
        body {
            background-color: #0f172a;
            color: #f8fafc;
            font-family: Arial, sans-serif;
            text-align: center;
            padding-top: 100px;
        }
        h1 { font-size: 80px; color: #f43f5e; margin: 0; }
        h2 { font-size: 24px; margin-top: 10px; }
        p { color: #94a3b8; font-size: 16px; }
        a {
            display: inline-block;
            margin-top: 20px;
            padding: 10px 24px;
            background-color: #2563eb;
            color: #fff;
            text-decoration: none;
            border-radius: 6px;
        }
    </style>
</head>
<body>
    <h1>404</h1>
    <h2>Không Tìm Thấy Trang Yêu Cầu</h2>
    <p>Đường dẫn bạn yêu cầu không tồn tại hoặc đã bị gỡ bỏ.</p>
    <a href="/">Quay về Trang chủ</a>
</body>
</html>
```

Phân quyền sở hữu cho thư mục web:
```bash
sudo chown -R www-data:www-data /var/www/my-web/html
sudo chmod -R 755 /var/www/my-web/html
```

---

### Bước 2: Cấu hình Server Block của Nginx

Tạo hoặc chỉnh sửa file cấu hình Server Block (ví dụ: `/etc/nginx/sites-available/my-web`):
```bash
sudo nano /etc/nginx/sites-available/my-web
```

Thêm nội dung cấu hình đầy đủ:
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

    # Cấu hình trang lỗi tùy biến 404
    error_page 404 /404.html;

    # Bảo vệ tệp tin lỗi bằng chỉ thị internal
    location = /404.html {
        root /var/www/my-web/html;
        internal;
    }
}
```

Kích hoạt site (tạo symlink vào `sites-enabled`) nếu là file mới:
```bash
sudo ln -sf /etc/nginx/sites-available/my-web /etc/nginx/sites-enabled/
```

---

### Bước 3: Kiểm tra cú pháp và Reload Nginx

Luôn kiểm tra cú pháp cấu hình Nginx trước khi áp dụng:
```bash
sudo nginx -t
```
*Kết quả mong đợi:*
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload lại dịch vụ Nginx mà không làm gián đoạn kết nối hiện tại:
```bash
sudo systemctl reload nginx
```

---

## 5. KIỂM TRA VÀ KẾT QUẢ XÁC MINH

### Kiểm tra 1: Truy cập một đường dẫn không tồn tại
```bash
curl -I http://<IP_ADDRESS_DROPLET>/invalid-path-demo
```
* **Kết quả:** Trả về mã trạng thái `HTTP/1.1 404 Not Found`.

Kiểm tra nội dung HTML trả về:
```bash
curl http://<IP_ADDRESS_DROPLET>/invalid-path-demo
```
* **Kết quả:** Trả về nội dung giao diện tùy biến của file `404.html` (chứa tiêu đề *"Không Tìm Thấy Trang Yêu Cầu"*).

---

### Kiểm tra 2: Truy cập trực tiếp vào tệp `/404.html`
```bash
curl -I http://<IP_ADDRESS_DROPLET>/404.html
```
* **Kết quả:** Vẫn trả về mã trạng thái `HTTP/1.1 404 Not Found` (thay vì `200 OK`) do chỉ thị `internal` đã chặn hoàn toàn việc gọi trực tiếp URL này từ bên ngoài.

---

## 6. HÌNH ẢNH MINH CHỨNG & LOG TERMINAL

### Hình ảnh minh chứng

![Cấu hình Nginx và kiểm tra curl test mã lỗi 404](./ex2-01.png)

---

### Trích xuất Log Terminal thực tế

```text
devops@0310263186:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

devops@0310263186:~$ sudo systemctl reload nginx

# 1. Truy cập đường dẫn không tồn tại:
devops@0310263186:~$ curl -I http://localhost/invalid-path-demo
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Date: Sun, 04 Oct 2026 12:25:00 GMT
Content-Type: text/html
Content-Length: 1450
Connection: keep-alive

# 2. Truy cập trực tiếp /404.html (Chặn bởi internal directive):
devops@0310263186:~$ curl -I http://localhost/404.html
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Date: Sun, 04 Oct 2026 12:25:05 GMT
Content-Type: text/html
Content-Length: 1450
Connection: keep-alive
```
