# Báo Cáo Thực Hành: Thiết Lập Thư Mục Web An Toàn Tại Phân Vùng Hệ Thống (/opt/)

- **Khóa học:** DevOps / Cloud Engineering
- **Bài tập:** Session 03 - Bài 5: Thiết lập thư mục Web an toàn tại phân vùng hệ thống (`/opt/`)
- **Đường dẫn nộp bài trên GitHub:** `homework/session_03/ex5/`
- **Tệp tin đính kèm:** `my-app.conf` (Nginx Server Block)
- **Học viên:** Hoàng Văn Lương
- **Hệ điều hành máy chủ:** Ubuntu 22.04 LTS (DigitalOcean Droplet)

---

## 1. Mục Tiêu Bài Tập

1. **Cô lập mã nguồn và logs:** Chuyển toàn bộ vị trí lưu trữ website tĩnh và hệ thống logs ra khỏi các đường dẫn mặc định (`/var/www/`, `/var/log/nginx/`) sang phân vùng `/opt/` (thư mục tiêu chuẩn cho các gói ứng dụng độc lập - Add-on Application Software Packages).
2. **Phân quyền nâng cao:** Phân định rõ ràng trách nhiệm giữa:
   - **Tài khoản `devops` (Owner):** Có toàn quyền đọc và ghi (Read/Write) để deploy, chỉnh sửa mã nguồn mà không cần sử dụng quyền `root` hay `sudo`.
   - **Nhóm `www-data` (Group của Nginx):** Có quyền đọc và thực thi (`r-x`) trên thư mục mã nguồn `html/` để phục vụ web tĩnh; đồng thời được cấp quyền ghi (`rwx`) vào thư mục `logs/` để tạo và ghi nhận log truy cập/lỗi trong thời gian thực.
3. **Cấu hình Nginx chuyên biệt:** Thiết lập Server Block tùy chỉnh trỏ Document Root tới `/opt/my-app/html/` và điều hướng ghi nhận logs vào `/opt/my-app/logs/`.
4. **Đảm bảo không phát sinh lỗi:** Nginx không bị lỗi `403 Forbidden` do thiếu quyền truy cập thư mục, và không bị lỗi `Permission denied` khi ghi log.

---

## 2. Thiết Kế Cấu Trúc & Bảng Phân Quyền Chi Tiết

### 2.1. Cấu Trúc Thư Mục Dự Án Tại `/opt/`

```text
/opt/my-app/
├── html/               # Chứa mã nguồn website tĩnh (index.html, css, js)
│   └── index.html
└── logs/               # Chứa nhật ký truy cập và nhật ký lỗi độc lập của dự án
    ├── access.log
    └── error.log
```

---

### 2.2. Ma Trận Phân Quyền (Permissions Matrix)

| Đường dẫn / Tệp tin | Loại | Quyền sở hữu (Owner:Group) | Quyền Octal (chmod) | Ký hiệu (Symbolic) | Giải thích mục đích |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `/opt/my-app/` | Thư mục gốc | `devops:www-data` | `755` (hoặc `750`) | `drwxr-xr-x` | Thư mục cha chứa dự án. User có full quyền, Group có quyền duyệt qua. |
| `/opt/my-app/html/` | Thư mục | `devops:www-data` | `755` (hoặc `750`) | `drwxr-xr-x` | Cần cờ `x` để Nginx (`www-data`) có thể duyệt vào đọc file. |
| `/opt/my-app/html/*` | Tệp tin | `devops:www-data` | `644` (hoặc `640`) | `-rw-r--r--` | `devops` đọc/ghi không cần sudo; Nginx chỉ đọc, không có cờ `x`. |
| `/opt/my-app/logs/` | Thư mục | `devops:www-data` | **`775`** (hoặc `770`) | `drwxrwxr-x` | **Bắt buộc `w` cho Group:** Nginx worker process chạy dưới `www-data` cần quyền tạo và ghi vào `access.log`, `error.log`. |

---

## 3. Các Bước Triển Khai Thực Hiện

### Bước 1: Khởi Tạo Cấu Trúc Thư Mục Dự Án Tại `/opt/`

Sử dụng tài khoản có quyền `sudo` để tạo cây thư mục tại `/opt/my-app/`:

```bash
# Tạo đồng thời thư mục html và logs
sudo mkdir -p /opt/my-app/html /opt/my-app/logs

# Tạo file index.html ban đầu để kiểm thử
sudo bash -c 'cat <<EOF > /opt/my-app/html/index.html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Production Web App - /opt/ Isolation</title>
</head>
<body>
    <h1>Hệ Thống Web Tĩnh Triển Khai Tại /opt/my-app/</h1>
    <p>Trạng thái: Hoạt động bình thường</p>
    <p>Quản trị viên: devops</p>
</body>
</html>
EOF'
```

---

### Bước 2: Phân Quyền Sở Hữu Đệ Quy Cho `devops:www-data`

Chuyển giao toàn bộ cây thư mục `/opt/my-app` cho user `devops` và group `www-data`:

```bash
sudo chown -R devops:www-data /opt/my-app
```

---

### Bước 3: Phân Quyền Truy Cập (chmod) Theo Tiêu Chuẩn Bảo Mật

Áp dụng quyền riêng biệt cho thư mục, tệp tin và thư mục logs:

```bash
# 1. Gán quyền 755 cho toàn bộ thư mục (đảm bảo quyền duyệt thư mục cho Nginx)
sudo find /opt/my-app -type d -exec chmod 755 {} +

# 2. Gán quyền 644 cho toàn bộ tệp tin bên trong
sudo find /opt/my-app -type f -exec chmod 644 {} +

# 3. CẤP QUYỀN GHI CHO GROUP www-data TẠI THƯ MỤC LOGS (chmod 775)
# Bước này cực kỳ quan trọng để Nginx có thể tạo và ghi file log
sudo chmod 775 /opt/my-app/logs
```

---

### Bước 4: Thiết Lập Server Block Cho Nginx

Tạo file cấu hình Virtual Host tại `/etc/nginx/sites-available/my-app.conf`:

```bash
sudo bash -c 'cat <<EOF > /etc/nginx/sites-available/my-app.conf
server {
    listen 80;
    server_name _;

    root /opt/my-app/html;
    index index.html index.htm;

    # Ghi nhận log trực tiếp vào phân vùng /opt/
    access_log /opt/my-app/logs/access.log;
    error_log /opt/my-app/logs/error.log warn;

    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOF'
```

Kích hoạt cấu hình và nạp lại dịch vụ Nginx:

```bash
# Kích hoạt site qua symbolic link
sudo ln -sf /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/

# Tắt cấu hình default cũ (nếu có)
sudo rm -f /etc/nginx/sites-enabled/default

# Kiểm tra cú pháp cấu hình Nginx
sudo nginx -t

# Nạp lại Nginx để áp dụng cấu hình mới
sudo systemctl reload nginx
```

---

## 4. Nhật Ký Kiểm Tra & Xác Thực (Verification Logs)

### 4.1. Kiểm Tra Phân Quyền Thư Mục Bằng Lệnh `ls -la`

Thực thi lệnh kiểm tra trạng thái quyền trên máy chủ:

```bash
devops@ubuntu-devops-server:~$ ls -la /opt/my-app/
total 16
drwxr-xr-x 4 devops www-data 4096 Oct  1 17:00 .
drwxr-xr-x 4 root   root     4096 Oct  1 16:55 ..
drwxr-xr-x 2 devops www-data 4096 Oct  1 17:01 html
drwxrwxr-x 2 devops www-data 4096 Oct  1 17:05 logs
```

Kiểm tra chi tiết bên trong thư mục `html/`:
```bash
devops@ubuntu-devops-server:~$ ls -la /opt/my-app/html/
total 12
drwxr-xr-x 2 devops www-data 4096 Oct  1 17:01 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 17:00 ..
-rw-r--r-- 1 devops www-data  348 Oct  1 17:01 index.html
```

> **Nhận xét kết quả:**
> - Toàn bộ tài nguyên thuộc sở hữu của `devops:www-data`.
> - Thư mục `logs` có quyền `drwxrwxr-x` (775), bảo đảm group `www-data` có quyền ghi (`w`).
> - Thư mục `html` có quyền `drwxr-xr-x` (755) và file `index.html` có quyền `-rw-r--r--` (644).

---

### 4.2. Thử Nghiệm Ghi File Bằng User `devops` Không Dùng `sudo`

Chuyển sang tài khoản `devops` và tiến hành cập nhật file:

```bash
# Chuyển phiên làm việc sang user devops
su - devops

# Thực hiện lệnh ghi thêm dữ liệu vào file index.html (KHÔNG dùng sudo)
echo "Update Test" >> /opt/my-app/html/index.html

# Kiểm tra mã trạng thái trả về ($? = 0 biểu thị thực thi thành công)
echo $?
0
```

Kiểm tra nội dung tệp tin sau khi ghi:
```bash
devops@ubuntu-devops-server:~$ tail -n 2 /opt/my-app/html/index.html
</html>
Update Test
```

---

### 4.3. Kiểm Tra Nginx Hoạt Động & Tự Động Ghi Log Vào `/opt/my-app/logs/`

Gửi request HTTP kiểm thử tới web server:

```bash
devops@ubuntu-devops-server:~$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Thu, 01 Oct 2026 17:10:00 GMT
Content-Type: text/html
Content-Length: 360
Last-Modified: Thu, 01 Oct 2026 17:08:15 GMT
Connection: keep-alive
ETag: "66fc722f-168"
Accept-Ranges: bytes
```

Kiểm tra nội dung file log truy cập (`access.log`) phát sinh trong thời gian thực:

```bash
devops@ubuntu-devops-server:~$ cat /opt/my-app/logs/access.log
127.0.0.1 - - [01/Oct/2026:17:10:00 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.81.0"
127.0.0.1 - - [01/Oct/2026:17:10:05 +0000] "GET / HTTP/1.1" 200 360 "-" "curl/7.81.0"
```

Kiểm tra trạng thái file log phát sinh trong thư mục:
```bash
devops@ubuntu-devops-server:~$ ls -la /opt/my-app/logs/
total 16
drwxrwxr-x 2 devops   www-data 4096 Oct  1 17:10 .
drwxr-xr-x 4 devops   www-data 4096 Oct  1 17:00 ..
-rw-r----- 1 www-data adm       186 Oct  1 17:10 access.log
-rw-r----- 1 www-data adm         0 Oct  1 17:05 error.log
```

> **Kết luận kiểm tra:**
> 1. Mã phản hồi trả về là `200 OK`, tuyệt đối không gặp lỗi `403 Forbidden`.
> 2. File `access.log` được Nginx tự động khởi tạo và ghi log thành công ngay bên trong `/opt/my-app/logs/`.

---

## 5. Phân Tích Chuyên Sâu Kiến Thức DevOps

### 5.1. Tiêu Chuẩn Phân Cấp Hệ Thống Tệp Tin (FHS) & Ý Nghĩa Của `/opt/`

- Theo chuẩn **Filesystem Hierarchy Standard (FHS)**:
  - `/var/www/`: Thư mục mặc định truyền thống của các web server như Apache, Nginx. Thường dễ bị nhầm lẫn giữa các dự án khác nhau và gắn liền với phân vùng gốc `/var`.
  - `/opt/` (Optional / Add-on): Là phân vùng chuyên dụng để cài đặt các ứng dụng của bên thứ ba hoặc các dự án phần mềm độc lập khép kín (self-contained application software packages).
- **Lợi ích khi đưa vào `/opt/`:**
  - **Cô lập hoàn toàn (Isolation):** Mã nguồn, tệp tin cấu hình và logs gom chung trong một cây thư mục duy nhất (`/opt/my-app/`), giúp việc backup, migration, hoặc mount phân vùng ổ cứng riêng (Dedicated EBS/Volume) trở nên cực kỳ đơn giản.
  - **Tách biệt khỏi hệ thống:** Không bị ảnh hưởng khi nâng cấp hoặc gỡ cài đặt các gói phần mềm mặc định của hệ điều hành.

---

### 5.2. Tại Sao Thư Mục `logs/` Bắt Buộc Group `www-data` Phải Có Quyền Ghi (`w`)?

1. Tiến trình Nginx khởi chạy gồm hai thành phần:
   - **Master Process:** Chạy dưới quyền `root` (để bind các cổng đặc quyền như port 80/443).
   - **Worker Processes:** Chạy dưới tài khoản không đặc quyền `www-data` (để xử lý các request HTTP từ client).
2. Khi mở hoặc tạo mới file log (hoặc khi logrotate thực hiện xoay vòng log), nếu thư mục `/opt/my-app/logs/` chỉ có quyền đọc `r-x`, worker process hoặc quá trình reload của Nginx sẽ không thể tạo file `access.log` mới và báo lỗi:
   ```text
   [alert] could not open error log file: open() "/opt/my-app/logs/error.log" failed (13: Permission denied)
   ```
3. Cấp quyền **`775`** (`drwxrwxr-x`) hoặc **`770`** (`drwxrwx---`) cho thư mục `logs/` với group `www-data` giải quyết triệt để vấn đề này mà vẫn đảm bảo tính an toàn cao nhất.

---

## 6. Kết Luận

- Đã hoàn thành toàn bộ yêu cầu của bài tập **Session 03 - Bài 5**.
- Mã nguồn và logs của website được cô lập thành công tại `/opt/my-app/`.
- Quản trị viên `devops` chỉnh sửa và cập nhật mã nguồn trực tiếp mà không cần dùng `sudo`.
- Nginx hoạt động ổn định, ghi log chính xác vào thư mục chỉ định, đảm bảo tuân thủ các quy tắc bảo mật hệ thống Linux trong môi trường Production.
