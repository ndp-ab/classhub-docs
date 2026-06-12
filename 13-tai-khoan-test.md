# 13 — Tài khoản test

> File này chỉ phục vụ phát triển + demo. **KHÔNG commit thông tin nhạy cảm thật** lên GitHub public.

## 13.1. Tài khoản dùng cho demo

> Tự đăng ký qua màn Signup trong app (chưa có DB seed tự động).

### Admin
| Trường | Giá trị |
|---|---|
| Họ tên | Nguyễn Duy Phong |
| Email | `admin@demo.com` |
| Mật khẩu | `12345678` |
| Vai trò | Admin của lớp K64KTPM3 |

### Member 1
| Trường | Giá trị |
|---|---|
| Họ tên | Trần Văn A |
| Email | `sv1@demo.com` |
| Mật khẩu | `12345678` |
| Vai trò | Member lớp K64KTPM3 |

### Member 2
| Trường | Giá trị |
|---|---|
| Họ tên | Lê Thị B |
| Email | `sv2@demo.com` |
| Mật khẩu | `12345678` |
| Vai trò | Member lớp K64KTPM3 |

### Tài khoản outsider (test cross-class)
| Trường | Giá trị |
|---|---|
| Họ tên | Outsider Test |
| Email | `outsider@demo.com` |
| Mật khẩu | `12345678` |
| Vai trò | Không tham gia lớp nào |

## 13.2. Lớp test

| Tên lớp | Khoa | Khóa | Invite Code | Admin | Members |
|---|---|---|---|---|---|
| K64KTPM3 | Công nghệ thông tin | K64 | `[Sinh ra khi tạo lớp]` | `admin@demo.com` | sv1, sv2 |

> **Note:** Invite code sinh ngẫu nhiên 6 ký tự uppercase mỗi lần tạo. Sau khi tạo cần ghi lại để gửi cho sv1, sv2 join.

## 13.3. Khoản thu test

| Title | Amount | Deadline | Trạng thái sau setup |
|---|---|---|---|
| Quỹ lớp tháng 5 | 50.000 đ | Ngày mai | sv1, sv2 chưa đóng (chip cam) |

## 13.4. Sự kiện test

| Title | Description | Location | Event time |
|---|---|---|---|
| Họp lớp cuối kỳ | Tổng kết hoạt động học kỳ 2 | P.A201 | Tuần sau, 18:00 |

## 13.5. Script setup nhanh trước demo

Sau khi BE đã chạy, mở Postman/cURL:

```bash
BASE=http://localhost:8080/api

# 1. Đăng ký 4 tài khoản
for u in admin sv1 sv2 outsider; do
  curl -s -X POST $BASE/auth/register \
    -H 'Content-Type: application/json' \
    -d "{\"fullName\":\"$u\",\"email\":\"$u@demo.com\",\"password\":\"12345678\"}"
done

# 2. Login Admin lấy token
ADMIN_TOKEN=$(curl -s -X POST $BASE/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin@demo.com","password":"12345678"}' | jq -r .token)

# 3. Admin tạo lớp
INVITE_CODE=$(curl -s -X POST $BASE/classrooms/create \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"className":"K64KTPM3","faculty":"CNTT","academicYear":"K64"}' \
  | jq -r .inviteCode)
echo "Invite code: $INVITE_CODE"

# 4. sv1, sv2 join lớp
for u in sv1 sv2; do
  TOKEN=$(curl -s -X POST $BASE/auth/login \
    -d "{\"email\":\"$u@demo.com\",\"password\":\"12345678\"}" \
    -H 'Content-Type: application/json' | jq -r .token)
  curl -s -X POST $BASE/classrooms/join \
    -H "Authorization: Bearer $TOKEN" \
    -H 'Content-Type: application/json' \
    -d "{\"inviteCode\":\"$INVITE_CODE\"}"
done

# 5. Sync bank catalog VietQR (bắt buộc trước khi cấu hình tài khoản lớp)
curl -s -X POST $BASE/banks/sync \
  -H "Authorization: Bearer $ADMIN_TOKEN"
curl -s $BASE/banks | jq '.[0:3]'

# 6. Admin cấu hình tài khoản ngân hàng cho lớp (bắt buộc trước khi tạo khoản thu)
CLASSROOM_ID=$(curl -s $BASE/classrooms/my \
  -H "Authorization: Bearer $ADMIN_TOKEN" | jq '.[0].id')
curl -s -X PUT $BASE/classrooms/$CLASSROOM_ID/bank-account \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"bankBin":"970422","accountNo":"0123456789","accountName":"NGUYEN VAN A","note":"Tài khoản thủ quỹ lớp"}'

# 7. Admin tạo khoản thu "Quỹ lớp tháng 5"
curl -s -X POST $BASE/fund/collections \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"classroomId\":$CLASSROOM_ID,\"title\":\"Quỹ lớp tháng 5\",\"amount\":50000,\"deadline\":\"2026-12-31\"}"
```

Sau script:
- 4 user đã đăng ký.
- 1 lớp K64KTPM3 đã có 3 thành viên (1 Admin + 2 Member).
- Bank catalog đã được sync từ VietQR; tài khoản ngân hàng test đã cấu hình cho lớp bằng `bankBin=970422`.
- 1 khoản thu 50.000đ với 2 payment chưa xác nhận.

→ Sẵn sàng demo từ Bước 6 trong `12-demo-script.md` (sinh viên xem QR).

## 13.6. Reset

Nếu cần demo lại từ đầu:
```sql
DROP DATABASE classhub_db;
CREATE DATABASE classhub_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
→ Restart BE → chạy lại script 13.5.

## 13.7. Tài khoản ngân hàng (cho QR thật)

> Tài khoản ngân hàng **không dùng config cố định trong `application.properties`**. Danh mục ngân hàng lấy từ bảng `banks`, được sync qua `POST /api/banks/sync`.

**Cách cấu hình đúng:**
1. Đăng nhập bằng tài khoản Admin.
2. Trong tab "Khoản thu" của lớp → bấm **"Thiết lập tài khoản nhận tiền"**.
3. Chọn ngân hàng từ danh mục backend, nhập số TK + chủ TK + ghi chú → bấm "Lưu".
4. QR sẽ tự động dùng tài khoản vừa cấu hình.

Hoặc dùng API trực tiếp (xem bước 5 trong script 13.5 phía trên).

> **Lưu ý:** Mỗi lớp có tài khoản riêng. Nếu đổi tài khoản, hệ thống giữ lịch sử (bản cũ `active=false`), QR luôn dùng tài khoản `active=true` hiện tại.
