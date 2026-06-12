# 11 — Hướng dẫn cài đặt và chạy project

## 11.1. Yêu cầu môi trường

| Phần | Phiên bản tối thiểu | Ghi chú |
|---|---|---|
| Java JDK | 17 | OpenJDK / Oracle JDK đều OK |
| Maven | 3.9+ | Đã có wrapper `mvnw` — không cần cài Maven toàn cục |
| MySQL | 8.0+ | |
| Flutter SDK | 3.10+ | `flutter --version` để check |
| Android Studio | Latest | Để có Android SDK + emulator |
| Git | bất kỳ | |
| RAM | 8 GB+ | Tối thiểu để chạy IDE + emulator + MySQL |

### Kiểm tra môi trường

```bash
java -version           # phải là 17+
mysql --version
flutter doctor          # Cho phép biết thiếu gì
git --version
```

## 11.2. Lấy source code

```bash
mkdir D:/big_dream && cd D:/big_dream

git clone https://github.com/agio7/classhub-api.git
git clone https://github.com/agio7/classhub-app.git
```

Cấu trúc cuối:
```
D:/big_dream/
├── classhub-api/    # Spring Boot backend
├── classhub_app/    # Flutter frontend
└── docs/            # Tài liệu (file này)
```

## 11.3. Cài đặt Backend

### 11.3.1. Tạo database MySQL

Login MySQL với root:
```sql
CREATE DATABASE classhub_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'classhub_user'@'localhost' IDENTIFIED BY 'ClassHub@2026';
GRANT ALL PRIVILEGES ON classhub_db.* TO 'classhub_user'@'localhost';
FLUSH PRIVILEGES;
```

> **Schema (13 bảng) sẽ được Hibernate auto-tạo** khi BE chạy lần đầu (`ddl-auto=update`).

### 11.3.2. Cấu hình `application.properties`

File: `classhub-api/src/main/resources/application.properties`

```properties
# DB — đổi nếu user/password khác
spring.datasource.url=jdbc:mysql://localhost:3306/classhub_db?useSSL=false&serverTimezone=Asia/Ho_Chi_Minh&allowPublicKeyRetrieval=true
spring.datasource.username=classhub_user
spring.datasource.password=ClassHub@2026

# JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Server
server.port=8080

# JWT — Có thể đổi secret, expiration tính bằng ms (86400000 = 24h)
jwt.secret=classhub-super-secret-key-2026-do-an-tot-nghiep-spring-boot-flutter
jwt.expiration=86400000

# VietQR
vietqr.template=compact2
vietqr.banks-url=https://api.vietqr.io/v2/banks

# Upload ảnh minh chứng — Đường dẫn thư mục lưu file (ngoài repo)
classhub.upload-dir=D:/big_dream/classhub-uploads
```

### 11.3.3. Chạy backend

**Lần đầu (build):**
```bash
cd D:/big_dream/classhub-api
./mvnw clean install -DskipTests
```

**Chạy server:**
```bash
./mvnw spring-boot:run
```

→ Console hiển thị: `Started ClasshubApiApplication in X seconds`
→ Server lắng nghe tại `http://localhost:8080`.

### 11.3.4. Kiểm tra backend hoạt động

```bash
# Test 1: API public — không cần token
curl -X POST http://localhost:8080/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"fullName":"Test","email":"test@a.com","password":"12345678"}'
# Expect: 200 với JWT

# Test 2: API authenticated — không token → phải 401
curl -i http://localhost:8080/api/classrooms/my
# Expect: HTTP/1.1 401
# Body: {"status":401,"message":"Chưa đăng nhập hoặc token không hợp lệ"}
```

### 11.3.5. Đồng bộ Bank catalog VietQR

Sau khi backend chạy, đồng bộ danh mục ngân hàng một lần trước khi cấu hình tài khoản nhận tiền cho lớp.

```bash
# Login lấy JWT
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@demo.com","password":"12345678"}' \
  | jq -r .token)

# Sync bank catalog từ VietQR vào bảng banks
curl -X POST http://localhost:8080/api/banks/sync \
  -H "Authorization: Bearer $TOKEN"

# Kiểm tra danh sách bank active (GET /api/banks hiện public)
curl http://localhost:8080/api/banks
```

`POST /api/banks/sync` hiện chỉ yêu cầu JWT vì project chưa có role system-admin/global-admin; production nên restrict hoặc chuyển thành job nội bộ.

Nếu cả 2 request OK → BE đã chạy đúng.

## 11.4. Cài đặt Frontend

### 11.4.1. Cài dependencies

```bash
cd D:/big_dream/classhub_app
flutter pub get
```

### 11.4.2. Cấu hình `baseUrl`

`baseUrl` được quản lý tập trung trong FE tại:

```txt
lib/core/config/app_config.dart
```

Mặc định là `http://localhost:8080/api`. Khi cần chạy trên môi trường khác,
truyền `API_BASE_URL` bằng `--dart-define`, không sửa tay từng service.

| Thiết bị | Cách chạy |
|---|---|
| Chrome web | `flutter run -d chrome` |
| Windows desktop | `flutter run -d windows` |
| Android emulator | `flutter run -d emulator-5554 --dart-define=API_BASE_URL=http://10.0.2.2:8080/api` |
| Thiết bị Android thật (cùng WiFi) | `flutter run --dart-define=API_BASE_URL=http://<IP-LAN>:8080/api` |

Build APK demo cho thiết bị thật:

```bash
flutter build apk --release --dart-define=API_BASE_URL=http://<IP-LAN>:8080/api
```

> Để biết IP LAN: chạy `ipconfig` (Windows) → tìm `IPv4 Address` ở `Wireless LAN adapter Wi-Fi`.

### 11.4.3. Chạy app

```bash
flutter devices             # liệt kê thiết bị available
flutter run                 # chạy trên thiết bị mặc định
# hoặc chỉ định:
flutter run -d chrome
flutter run -d emulator-5554
```

### 11.4.4. Build APK release (cho demo offline)

```bash
flutter build apk --release
# File output: build/app/outputs/flutter-apk/app-release.apk
```

## 11.5. Cấu hình firewall (nếu test trên device thật)

Windows Firewall mặc định chặn inbound port 8080.

**Cách 1: Tắt firewall tạm thời (demo nhanh):**
- Control Panel → Windows Defender Firewall → Turn off (chỉ Private network).

**Cách 2 (an toàn hơn): Mở port 8080:**
- Control Panel → Windows Defender Firewall → Advanced Settings → Inbound Rules → New Rule → Port → TCP 8080 → Allow.

## 11.6. Troubleshoot

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| BE: `Access denied for user 'classhub_user'` | Sai user/password MySQL | Kiểm tra `application.properties` |
| BE: `Communications link failure` | MySQL chưa chạy | `net start MySQL80` (Windows) hoặc kiểm tra service |
| BE: port 8080 đã dùng | Process khác chiếm port | `netstat -ano \| findstr :8080` → kill PID, hoặc đổi `server.port=8081` |
| FE: `flutter pub get` lỗi | Mạng / version conflict | `flutter clean && flutter pub get` |
| FE: app không kết nối được BE | baseUrl sai cho thiết bị | Xem mục 11.4.2 |
| FE: 401 dù vừa login | JWT secret BE thay đổi sau khi login | Logout → login lại trên FE |
| FE Android emulator: `Connection refused` | Dùng `localhost` (sai) | Chạy với `--dart-define=API_BASE_URL=http://10.0.2.2:8080/api` |
| Hibernate: `Unknown column 'confirmed_by'` | Đã upgrade code B3 nhưng MySQL chưa ALTER | `ddl-auto=update` phải bật; restart BE |
| BE: Ảnh upload 400 "File phải là ảnh" | FE không set `contentType` trong `MultipartFile.fromPath` | FE thêm `contentType: MediaType('image', 'jpeg')` |
| BE: `upload-dir` không tồn tại | Thư mục `classhub-uploads` chưa tạo | Tạo thủ công: `mkdir D:/big_dream/classhub-uploads` |

## 11.7. Reset / clean data

**Xoá toàn bộ DB và làm lại:**
```sql
DROP DATABASE classhub_db;
CREATE DATABASE classhub_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
Sau đó restart BE → Hibernate tự tạo lại schema từ entity.

**Xoá SharedPreferences trên Flutter:**
- Trong app: bấm Logout (gọi `prefs.clear()`).
- Hoặc gỡ cài đặt app rồi cài lại.

## 11.8. Tài khoản test

Xem `13-tai-khoan-test.md`.

## 11.9. Link tham khảo

- Spring Boot 4 docs: https://docs.spring.io/spring-boot/4.0.5/reference/
- Flutter docs: https://docs.flutter.dev
- VietQR: https://www.vietqr.io
- jjwt: https://github.com/jwtk/jjwt
