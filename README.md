# Apache Guacamole Docker Stack

Cổng điều khiển máy tính và máy chủ từ xa hoàn toàn qua trình duyệt web (Clientless Remote Desktop Gateway), hỗ trợ giao thức RDP, SSH, VNC, và Telnet. Tích hợp cơ sở dữ liệu PostgreSQL 16 Alpine, xác thực 2 lớp TOTP, tự động sinh bảng dữ liệu, và tự động cập nhật qua Watchtower.

---

## 🚀 Các dịch vụ trong Stack

| Dịch vụ | Image | Cổng Host | Chức năng |
|---|---|---|---|
| **guac-initdb** | `guacamole/guacamole:${GUAC_VERSION}` | *Không mở* | Tự động sinh file SQL schema (`./init/initdb.sql`) rồi tự thoát |
| **postgres** | `postgres:16-alpine` | *Không mở* | Cơ sở dữ liệu lưu người dùng, danh sách máy và lịch sử phiên |
| **guacd** | `guacamole/guacd:${GUAC_VERSION}` | *Không mở* | Daemon biên dịch giao thức RDP/SSH/VNC |
| **guacamole** | `guacamole/guacamole:${GUAC_VERSION}` | `${GUAC_PORT}:8080` | Giao diện web người dùng HTML5 |
| **watchtower** | `containrrr/watchtower:latest` | *Không mở* | Tự động cập nhật Guacamole lúc 4:00 sáng hàng ngày |

---

## 📁 Cấu trúc thư mục

```text
.
├── .gitignore               # Loại trừ .env chứa mật khẩu thật
├── .env.example             # Mẫu cấu hình biến môi trường
├── docker-compose.yml       # Khai báo stack dịch vụ Guacamole
├── init/
│   └── initdb.sql           # Schema cơ sở dữ liệu do guac-initdb tự sinh
├── nginx/
│   └── guacamole.conf       # File mẫu cấu hình Nginx Reverse Proxy
└── README.md                # Tài liệu hướng dẫn này
```

---

## 🛠️ Hướng dẫn triển khai

### 1. Khởi tạo biến môi trường (.env)
Tạo file `.env` từ file mẫu:
```bash
cp .env.example .env
```

Chỉnh sửa các thông số trong `.env`:
- `POSTGRES_PASSWORD`: Đặt mật khẩu mạnh cho PostgreSQL.
- `GUAC_PORT`: Cổng truy cập Web trên máy host (mặc định: `8005`).
- `TOTP_ENABLED`: Đặt `true` nếu muốn bật xác thực 2 bước (Google Authenticator) khi đăng nhập.

### 2. Khởi chạy Stack
```bash
docker compose up -d
```

### 3. Cơ chế hoạt động của Stack
- Khi chạy lần đầu, container `guac-initdb` kiểm tra nếu `./init/initdb.sql` chưa có thì sẽ trích xuất schema PostgreSQL từ Guacamole ra file, sau đó container tự thoát hoàn toàn.
- Container `postgres` sử dụng Docker Named Volume `guac_db` để đảm bảo quyền truy cập dữ liệu ổn định và nạp schema từ `./init/initdb.sql`.
- Container `guacamole` web chỉ bắt đầu chạy khi `postgres` đã hoàn tất khởi tạo và chuyển sang trạng thái `healthy`.

### 4. Đăng nhập lần đầu
- **URL:** `http://<IP-Server>:8005/`
- **Username mặc định:** `guacadmin`
- **Password mặc định:** `guacadmin`
- ⚠️ **Lưu ý bảo mật:** Vào mục **Settings -> Users -> guacadmin** và đổi mật khẩu quản trị ngay lập tức sau khi đăng nhập lần đầu.

---

## 🌐 Cấu hình Nginx Reverse Proxy
File mẫu cấu hình có sẵn tại `nginx/guacamole.conf`. Các điểm cốt lõi:
- **Tắt buffer (bắt buộc cho remote desktop):**
  ```nginx
  proxy_buffering         off;
  proxy_request_buffering off;
  proxy_cache             off;
  ```
- **Hỗ trợ WebSocket:**
  ```nginx
  proxy_set_header Upgrade    $http_upgrade;
  proxy_set_header Connection $http_connection;
  ```
- **Tăng thời gian timeout cho phiên làm việc:**
  ```nginx
  proxy_connect_timeout 10s;
  proxy_send_timeout    3600s;
  proxy_read_timeout    3600s;
  ```
- **Cho phép truyền file lớn:** `client_max_body_size 2g;`

---

## 🔄 Tự động cập nhật với Watchtower
Container Watchtower được gắn nhãn độc lập:
```yaml
labels:
  - com.centurylinklabs.watchtower.scope=guacamole
```
- Lịch trình: Kiểm tra vào **4:00 sáng hàng ngày** (`0 0 4 * * *`).
- Phạm vi: Chỉ cập nhật các service có scope `guacamole` (postgres, guacd, guacamole).
- Tự động xóa image cũ sau khi cập nhật (`--cleanup`).
