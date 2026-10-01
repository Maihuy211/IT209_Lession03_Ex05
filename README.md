# Bài 5: Thiết lập thư mục Web an toàn tại /opt/

## 1. Mục tiêu
Cô lập mã nguồn tĩnh và logs ra khỏi đường dẫn mặc định sang phân vùng `/opt/`:
- Mã nguồn: `/opt/my-app/html/`
- Logs: `/opt/my-app/logs/` (`access.log`, `error.log`)
- Ownership đệ quy: `devops:www-data`
- `devops` (quản trị thường) đọc/ghi không cần sudo; `www-data` (tiến trình Nginx) đọc + thực thi web tĩnh và ghi được vào `logs/`.
- Nginx `access_log` + `error_log` ghi trực tiếp vào `/opt/my-app/logs/`.

Verify thực tế trên **Ubuntu 22.04 + Nginx 1.18 trong Docker** bằng các lệnh thủ công dưới đây.

## 2. Cấu trúc nộp bài
```
bai5/
├── my-app.conf   # Server Block Nginx nộp chính
└── README.md     # Báo cáo này (log kiểm tra + trạng thái phân quyền)
```

## 3. Các lệnh đã chạy trên máy chủ Ubuntu (toàn bộ làm thủ công, không script phụ)
```bash
sudo apt update && sudo apt install -y nginx
id devops 2>/dev/null || sudo useradd -m -s /bin/bash devops
sudo usermod -aG www-data devops
sudo mkdir -p /opt/my-app/html /opt/my-app/logs
sudo touch /opt/my-app/logs/access.log /opt/my-app/logs/error.log
sudo chown -R devops:www-data /opt/my-app
sudo chmod 755 /opt/my-app /opt/my-app/html
sudo chmod 775 /opt/my-app/logs
sudo chmod g+s /opt/my-app/logs || true
sudo chmod 664 /opt/my-app/logs/access.log /opt/my-app/logs/error.log
# Triển khai web tĩnh mẫu + phân quyền file web:
# (nội dung index.html xem mục 5.5, kết quả curl)
sudo chmod 644 /opt/my-app/html/index.html
sudo chown devops:www-data /opt/my-app/html/index.html
# Deploy Server Block:
sudo cp my-app.conf /etc/nginx/sites-available/my-app
sudo ln -sf /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/my-app
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo nginx -s reload || (sudo nginx -s stop; sleep 1; sudo nginx)
# Kiểm tra nhanh theo đề bài:
ls -la /opt/my-app/
su devops -c 'echo "Update Test" >> /opt/my-app/html/index.html'
curl -I http://localhost/
tail -n 20 /opt/my-app/logs/access.log
```

## 4. File cấu hình Nginx nộp (`my-app.conf`)
```nginx
server {
    listen 80 default_server;
    server_name _;

    root /opt/my-app/html;
    index index.html;

    # Logs cô lập ra /opt theo yêu cầu Bài 5
    access_log /opt/my-app/logs/access.log combined;
    error_log  /opt/my-app/logs/error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Triển khai: `/etc/nginx/sites-available/my-app` → symlink `/etc/nginx/sites-enabled/my-app`, đã xóa `default`.
Kiểm tra `nginx -T` thấy đúng 3 dòng quan trọng:
```
root /opt/my-app/html;
access_log /opt/my-app/logs/access.log combined;
error_log  /opt/my-app/logs/error.log warn;
```

## 5. Bằng chứng kiểm tra (copy nguyên văn từ máy chủ test)

### 5.1. Phân quyền sở hữu `ls -la /opt/my-app/`
```text
$ ls -la /opt/my-app/
total 16
drwxr-xr-x 4 devops www-data 4096 Oct  1 03:01 .
drwxr-xr-x 1 root   root     4096 Oct  1 03:01 ..
drwxr-xr-x 2 devops www-data 4096 Oct  1 03:01 html
drwxrwsr-x 2 devops www-data 4096 Oct  1 03:01 logs
```
Đạt: toàn bộ `/opt/my-app/` thuộc `devops:www-data`.

### 5.2. Chi tiết số octal + user/group (`stat`, `id`, `namei`)
```text
$ stat -c '%a %U:%G %n' /opt/my-app /opt/my-app/html /opt/my-app/html/index.html /opt/my-app/logs /opt/my-app/logs/access.log /opt/my-app/logs/error.log
755 devops:www-data /opt/my-app
755 devops:www-data /opt/my-app/html
644 devops:www-data /opt/my-app/html/index.html
2775 devops:www-data /opt/my-app/logs
664 devops:www-data /opt/my-app/logs/access.log
664 devops:www-data /opt/my-app/logs/error.log

$ id devops; id www-data
uid=1000(devops) gid=1000(devops) groups=1000(devops),33(www-data)
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ namei -l /opt/my-app/html/index.html
f: /opt/my-app/html/index.html
drwxr-xr-x root   root     /
drwxr-xr-x root   root     opt
drwxr-xr-x devops www-data my-app
drwxr-xr-x devops www-data html
-rw-r--r-- devops www-data index.html
```
Giải thích:
- Thư mục web `755`, file web `644` → `devops` rwx/rw, group `www-data` rx/r, đúng khung `750/755` + `640/644` của đề.
- Thư mục `logs/` `2775` (= `775` + `setgid`) → group `www-data` có `rwx`, ghi log được (đúng gợi ý `chmod 775 /opt/my-app/logs/`).
- File log `664` → group `www-data` ghi được.
- `devops` đã là thành viên `www-data`.

### 5.3. Ghi thử bằng user `devops` KHÔNG dùng sudo
```text
$ su devops -c 'echo "Update Test" >> /opt/my-app/html/index.html && echo OK-ghi-thanh-cong'
OK-ghi-thanh-cong

$ tail -n 5 /opt/my-app/html/index.html
  <h1>My App - Bai 5 OK</h1>
  <p>Thu muc web an toan tai /opt/my-app/html (owner devops:www-data).</p>
</body>
</html>
Update Test

$ ls -l /opt/my-app/html/
total 4
-rw-r--r-- 1 devops www-data 319 Oct  1 03:02 index.html
```
Đạt: ghi thành công không cần `sudo`, không `Permission denied`.

### 5.4. Tiến trình Nginx chạy bằng `www-data` + `www-data` ghi được vào `logs/`
```text
$ ps aux | grep nginx
root        3146  ...  nginx: master process nginx
www-data    3147  ...  nginx: worker process
www-data    3148  ...  nginx: worker process
...

$ grep -R "^user" /etc/nginx/nginx.conf
user www-data;

$ su www-data -s /bin/sh -c 'touch /opt/my-app/logs/wwwdata-write-test.tmp && echo www-data-write-OK && ls -l /opt/my-app/logs/'
www-data-write-OK
total 4
-rw-rw-r-- 1 devops   www-data 361 Oct  1 03:03 access.log
-rw-rw-r-- 1 devops   www-data   0 Oct  1 03:01 error.log
-rw-rw-r-- 1 www-data www-data   0 Oct  1 03:03 wwwdata-write-test.tmp
```
Đạt: worker `www-data`, group có quyền ghi `logs/` nên Nginx tự ghi log được (file test đã xóa sau kiểm tra).

### 5.5. Nginx hoạt động bình thường (200, không 403) + tự ghi log
```text
$ curl -s -o /dev/null -w "HTTP code: %{http_code}\n" http://localhost/
HTTP code: 200

$ curl -s http://localhost/ | head -n 20
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>My App - Bai 5</title>
</head>
<body>
  <h1>My App - Bai 5 OK</h1>
  <p>Thu muc web an toan tai /opt/my-app/html (owner devops:www-data).</p>
</body>
</html>
Update Test

$ curl -s -o /dev/null -w "404-page code: %{http_code}\n" http://localhost/khong-ton-tai-xyz
404-page code: 404

$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Trang 404 trả `404` (không phải `403`) → không lỗi phân quyền.

### 5.6. Nội dung logs phát sinh (`tail /opt/my-app/logs/access.log`)
```text
$ cat /opt/my-app/logs/access.log
127.0.0.1 - - [01/Oct/2026:03:01:26 +0000] "GET / HTTP/1.1" 200 307 "-" "curl/7.81.0"
127.0.0.1 - - [01/Oct/2026:03:02:33 +0000] "GET / HTTP/1.1" 200 319 "-" "curl/7.81.0"
127.0.0.1 - - [01/Oct/2026:03:02:33 +0000] "GET / HTTP/1.1" 200 319 "-" "curl/7.81.0"
127.0.0.1 - - [01/Oct/2026:03:03:02 +0000] "GET /khong-ton-tai-xyz HTTP/1.1" 404 162 "-" "curl/7.81.0"

$ cat /opt/my-app/logs/error.log
(empty — không có lỗi, là trạng thái tốt)
```
Đạt: mỗi `curl` đều sinh một dòng `access.log` trong `/opt/my-app/logs/`, đúng `combined` format; `error.log` tồn tại và Nginx có quyền ghi (đang rỗng vì không lỗi).

### 5.7. Liệt kê đầy đủ `ls -laR /opt/my-app/`
```text
/opt/my-app/:
total 16
drwxr-xr-x 4 devops www-data 4096 Oct  1 03:01 .
drwxr-xr-x 1 root   root     4096 Oct  1 03:01 ..
drwxr-xr-x 2 devops www-data 4096 Oct  1 03:01 html
drwxrwsr-x 2 devops www-data 4096 Oct  1 03:01 logs

/opt/my-app/html:
total 12
drwxr-xr-x 2 devops www-data 4096 Oct  1 03:01 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 03:01 ..
-rw-r--r-- 1 devops www-data  319 Oct  1 03:02 index.html

/opt/my-app/logs:
total 12
drwxrwsr-x 2 devops www-data 4096 Oct  1 03:01 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 03:01 ..
-rw-rw-r-- 1 devops www-data  361 Oct  1 03:03 access.log
-rw-rw-r-- 1 devops www-data    0 Oct  1 03:01 error.log
```

## 6. Kết luận đối chiếu yêu cầu
| Yêu cầu | Trạng thái |
|---|---|
| `/opt/my-app/html/` chứa mã nguồn | ✅ Đạt, `index.html` serve 200 |
| `/opt/my-app/logs/` riêng | ✅ Đạt, `access.log` + `error.log` |
| `chown -R devops:www-data /opt/my-app` | ✅ Đạt, `ls -la` + `stat` chứng minh |
| Thư mục `750/755`, file `640/644`; `logs/` cho group `www-data` ghi | ✅ Đạt, `755/644` + `775/664` |
| `access_log` + `error_log` vào `/opt` | ✅ Đạt, `nginx -T` + log thực tế |
| `ls -la`, `echo >> index.html` bằng `devops` không sudo, `tail access.log` | ✅ Đạt, logs mục 5.1/5.3/5.6 |
| Không `403`, tự ghi log | ✅ Đạt, `200` + `404` (không `403`), log tăng sau mỗi request |

## 7. Lưu ý khi chấm trên máy thật
- Nếu `error.log` báo `permission denied` khi reload, chạy lại 2 dòng gợi ý của đề: `chown -R devops:www-data /opt/my-app && chmod 775 /opt/my-app/logs` rồi `nginx -s reload`.
- Nếu SELinux/AppArmor chặn, cần thêm rule cho phép `httpd`/`nginx` đọc `/opt`, nhưng trên Ubuntu mặc định + Docker test ở trên thì không cần.
