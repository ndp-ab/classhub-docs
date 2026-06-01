# 07 — API Documentation

Tài liệu mô tả 21 endpoint REST của ClassHub Backend.

**Base URL:** `http://localhost:8080/api` (đổi theo môi trường)
**Auth:** mọi endpoint trừ `/api/auth/**` yêu cầu header:
```
Authorization: Bearer <jwt_token>
```

## 7.1. Quy ước chung

### Response thành công
- HTTP **200 OK** + body JSON (hoặc **204 No Content** với `DELETE`).

### Response lỗi
| HTTP | Khi nào | Body |
|---|---|---|
| 400 | Validate fail, nghiệp vụ sai | `{"status":400, "message":"..."}` hoặc `{"status":400, "message":"Dữ liệu không hợp lệ", "errors":{"field":"msg"}}` |
| 401 | Chưa đăng nhập hoặc token sai/hết hạn | `{"status":401, "message":"Chưa đăng nhập hoặc token không hợp lệ"}` |
| 403 | Đã đăng nhập nhưng không có quyền | `{"status":403, "message":"..."}` |
| 500 | Lỗi server | `{"status":500, "message":"Lỗi máy chủ"}` |

---

## 7.2. Auth — `/api/auth`

### POST /api/auth/register
Đăng ký tài khoản mới. **Public** (không cần token).

**Request body:**
```json
{
  "fullName": "Nguyễn Duy Phong",
  "email": "phong@example.com",
  "password": "12345678"
}
```

**Response 200:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "userId": 7,
  "fullName": "Nguyễn Duy Phong",
  "email": "phong@example.com"
}
```

**Lỗi:**
- 400 Email đã tồn tại / Email không đúng format / Mật khẩu rỗng.

### POST /api/auth/login
Đăng nhập. **Public**.

**Request:** `{"email":"...", "password":"..."}`
**Response:** giống register.
**Lỗi:** 400 Email không tồn tại / Mật khẩu không đúng.

---

## 7.3. Classroom — `/api/classrooms`

### POST /api/classrooms/create
Tạo lớp mới. Người tạo tự động là **Admin**.

**Request:**
```json
{
  "className": "64KTPM3",
  "faculty": "Công nghệ thông tin",
  "academicYear": "K64"
}
```

**Response:**
```json
{
  "id": 12,
  "className": "64KTPM3",
  "faculty": "Công nghệ thông tin",
  "academicYear": "K64",
  "inviteCode": "A3B7K9",
  "role": "ADMIN"
}
```

### POST /api/classrooms/join
Tham gia lớp bằng mã mời. Hệ thống **tự sinh payment bổ sung** cho mọi đợt thu đã tồn tại của lớp đó.

**Request:** `{"inviteCode": "A3B7K9"}`
**Response:** giống `/create` nhưng `role` = `"MEMBER"`.
**Lỗi:** 400 Mã không hợp lệ / Đã tham gia lớp này rồi.

### GET /api/classrooms/my
Lấy danh sách lớp user đã tham gia.

**Response:**
```json
[
  {
    "id": 12,
    "className": "64KTPM3",
    "faculty": "Công nghệ thông tin",
    "academicYear": "K64",
    "inviteCode": "A3B7K9",
    "role": "ADMIN"
  }
]
```

---

## 7.4. Quỹ — Khoản thu — `/api/fund/collections`

### POST /api/fund/collections
Tạo đợt thu mới. **Chỉ Admin của lớp**. Hệ thống tự sinh `FundPayment` cho all thành viên.

**Request:**
```json
{
  "classroomId": 12,
  "title": "Quỹ lớp tháng 5",
  "amount": 50000,
  "deadline": "2026-05-31"
}
```

**Validate:**
- `title` không rỗng (`@NotBlank`)
- `amount >= 0.01` (`@DecimalMin`)
- `classroomId` không null

**Response:**
```json
{
  "id": 42,
  "title": "Quỹ lớp tháng 5",
  "amount": 50000,
  "deadline": "2026-05-31",
  "createdByName": "Admin A",
  "totalMembers": 30,
  "paidCount": 0,
  "createdAt": "2026-05-15T22:00:00"
}
```

**Lỗi:**
- 400 Validation fail (kèm field cụ thể).
- 403 User không phải Admin của lớp.

### GET /api/fund/collections/{classroomId}
Lấy danh sách đợt thu của 1 lớp. **Member của lớp**.

**Response:** mảng `CollectionResponse`.
**Lỗi:** 403 Bạn không thuộc lớp này.

### GET /api/fund/collections/{collectionId}/payments
Admin xem ai đã đóng / chưa.

**Response:**
```json
[
  {
    "id": 100,
    "userId": 8,
    "fullName": "Sinh viên B",
    "collectionTitle": "Quỹ lớp tháng 5",
    "amount": 50000,
    "deadline": "2026-05-31",
    "isPaid": false,
    "confirmedByAdmin": false,
    "paidAt": null,
    "confirmedByName": null
  },
  {
    "id": 101,
    "userId": 9,
    "fullName": "Sinh viên C",
    "collectionTitle": "Quỹ lớp tháng 5",
    "amount": 50000,
    "deadline": "2026-05-31",
    "isPaid": true,
    "confirmedByAdmin": true,
    "paidAt": "2026-05-16T10:30:00",
    "confirmedByName": "Admin A"
  }
]
```

**Lỗi:** 403 Bạn không phải Admin của lớp này.

### PUT /api/fund/payments/{paymentId}/confirm
Admin xác nhận sinh viên đã đóng.

**Request body:** rỗng.
**Response:** 1 `PaymentResponse` đã cập nhật (gồm `confirmedByName`, `paidAt`, `status="CONFIRMED"`).
**Lỗi:**
- 403 Không phải admin của lớp.
- 400 "Khoản thu này đã được xác nhận" (idempotency).

### POST /api/fund/payments/{paymentId}/mark-paid  *(GP1)*
**Member** tự báo "Tôi đã chuyển khoản" sau khi CK qua app ngân hàng.
Chuyển payment từ `UNPAID` sang `PENDING_VERIFICATION`.

**Request body:** rỗng.
**Response:** `PaymentResponse` với `status="PENDING_VERIFICATION"`, `markedPaid=true`, `markedPaidAt` được set.
**Lỗi:**
- 403 "Bạn chỉ có thể báo chuyển khoản cho khoản đóng của chính mình" (không phải owner).
- 400 "Bạn đã báo chuyển khoản trước đó, vui lòng chờ Admin xác nhận" (idempotency).
- 400 "Khoản này đã được Admin xác nhận, không cần báo lại".

### GET /api/fund/payments/my/{classroomId}
Sinh viên xem nợ cá nhân trong 1 lớp.

**Response:** mảng `PaymentResponse` của riêng user.
**Lỗi:** 403 Bạn không thuộc lớp này.

### GET /api/fund/payments/{paymentId}/qr
Lấy QR thanh toán. **Chỉ chủ payment** mới gọi được.

**Response:**
```json
{
  "paymentId": 100,
  "qrUrl": "https://img.vietqr.io/image/970415-109875610620-compact2.png?amount=50000&addInfo=QUY42-SV8-1715783000&accountName=Nguyen+Duy+Phong",
  "amount": 50000,
  "paymentCode": "QUY42-SV8-1715783000",
  "collectionTitle": "Quỹ lớp tháng 5",
  "deadline": "2026-05-31"
}
```

**Lỗi:** 403 "Bạn chỉ có thể xem QR của khoản đóng của chính mình".

### GET /api/fund/payments/{paymentId}/status
Polling trạng thái — Flutter gọi mỗi 5s. **3 trạng thái (GP1):**
- `UNPAID` — Member chưa làm gì
- `PENDING_VERIFICATION` — Member đã báo CK, chờ Admin
- `CONFIRMED` — Admin đã xác nhận

**Response khi UNPAID:**
```json
{
  "paymentId": 100,
  "status": "UNPAID",
  "markedPaid": false,
  "markedPaidAt": null,
  "confirmedByAdmin": false,
  "paidAt": null,
  "paymentCode": "QUY42-SV8-1715783000",
  "isPaid": false
}
```

**Sau khi Member bấm "Tôi đã CK":**
```json
{
  "paymentId": 100,
  "status": "PENDING_VERIFICATION",
  "markedPaid": true,
  "markedPaidAt": "2026-05-16T10:15:00",
  "confirmedByAdmin": false,
  "paidAt": null,
  "paymentCode": "QUY42-SV8-1715783000",
  "isPaid": false
}
```

**Sau khi Admin confirm:**
```json
{
  "paymentId": 100,
  "status": "CONFIRMED",
  "markedPaid": true,
  "markedPaidAt": "2026-05-16T10:15:00",
  "confirmedByAdmin": true,
  "paidAt": "2026-05-16T10:30:00",
  "paymentCode": "QUY42-SV8-1715783000",
  "isPaid": true
}
```

---

## 7.5. Quỹ — Khoản chi — `/api/fund/expenses`

### POST /api/fund/expenses
Admin tạo khoản chi.

**Request:**
```json
{
  "classroomId": 12,
  "title": "Mua nước cho buổi liên hoan",
  "amount": 200000,
  "reason": "Liên hoan cuối kỳ ngày 30/4"
}
```

**Response:**
```json
{
  "id": 50,
  "title": "Mua nước cho buổi liên hoan",
  "amount": 200000,
  "reason": "Liên hoan cuối kỳ ngày 30/4",
  "createdByName": "Admin A",
  "createdAt": "2026-05-15T22:30:00"
}
```

### GET /api/fund/expenses/{classroomId}
Member của lớp xem danh sách chi.
Response: mảng `ExpenseResponse`.

---

## 7.6. Sự kiện — `/api/events`

### POST /api/events
Admin tạo sự kiện.

**Request:**
```json
{
  "classroomId": 12,
  "title": "Họp lớp cuối kỳ",
  "description": "Tổng kết hoạt động học kỳ 2",
  "location": "P.A201",
  "eventTime": "2026-05-30T18:00:00"
}
```

**Response:**
```json
{
  "id": 20,
  "title": "Họp lớp cuối kỳ",
  "description": "Tổng kết hoạt động học kỳ 2",
  "location": "P.A201",
  "eventTime": "2026-05-30T18:00:00",
  "createdByName": "Admin A",
  "volunteerCount": 0,
  "checkedInCount": 0,
  "createdAt": "2026-05-15T22:35:00"
}
```

### GET /api/events/{classroomId}
Member của lớp xem danh sách sự kiện. Sắp xếp theo `event_time DESC`.

### POST /api/events/{eventId}/volunteer
Sinh viên đăng ký tham gia.

**Request body:** rỗng.
**Response:** `EventParticipantResponse`:
```json
{
  "id": 200,
  "eventId": 20,
  "userId": 8,
  "fullName": "Sinh viên B",
  "eventTitle": "Họp lớp cuối kỳ",
  "checkedIn": false,
  "checkedInAt": null,
  "checkedByName": null,
  "registeredAt": "2026-05-16T08:00:00"
}
```

**Lỗi:**
- 400 "Bạn đã đăng ký tham gia sự kiện này rồi".
- 403 Không thuộc lớp của sự kiện.

### DELETE /api/events/{eventId}/volunteer
Sinh viên huỷ đăng ký.

**Response:** 204 No Content.
**Lỗi:**
- 400 Chưa đăng ký.
- 400 "Không thể hủy đăng ký sau khi đã check-in".

### GET /api/events/{eventId}/participants
Admin xem danh sách người đăng ký.
**Response:** mảng `EventParticipantResponse`.

### PUT /api/events/{eventId}/checkin/{userId}
Admin check-in cho sinh viên.

**Request body:** rỗng.
**Response:** `EventParticipantResponse` đã cập nhật, kèm `checkedByName`.
**Lỗi:**
- 400 "Sinh viên này đã được check-in rồi" (idempotency).
- 400 "Sinh viên chưa đăng ký sự kiện này".
- 403 Không phải Admin của lớp.

### GET /api/events/my/{classroomId}
Sinh viên xem các sự kiện mình đã đăng ký trong 1 lớp.
**Response:** mảng `EventParticipantResponse`.

---

## 7.7. Bảng tổng hợp

| # | Method | Endpoint | Quyền | Mục đích |
|---|---|---|---|---|
| 1 | POST | `/api/auth/register` | Public | Đăng ký |
| 2 | POST | `/api/auth/login` | Public | Đăng nhập |
| 3 | POST | `/api/classrooms/create` | Authenticated | Tạo lớp |
| 4 | POST | `/api/classrooms/join` | Authenticated | Join lớp |
| 5 | GET | `/api/classrooms/my` | Authenticated | List lớp của mình |
| 6 | POST | `/api/fund/collections` | Admin lớp | Tạo đợt thu |
| 7 | GET | `/api/fund/collections/{classroomId}` | Member lớp | List đợt thu |
| 8 | GET | `/api/fund/collections/{collectionId}/payments` | Admin lớp | Ai đóng/chưa |
| 9 | PUT | `/api/fund/payments/{paymentId}/confirm` | Admin lớp | Xác nhận (PENDING → CONFIRMED) |
| 9b | POST | `/api/fund/payments/{paymentId}/mark-paid` | Owner | **GP1: Member báo đã CK (UNPAID → PENDING_VERIFICATION)** |
| 10 | GET | `/api/fund/payments/my/{classroomId}` | Member lớp | Nợ cá nhân |
| 11 | GET | `/api/fund/payments/{paymentId}/qr` | Owner | Lấy QR |
| 12 | GET | `/api/fund/payments/{paymentId}/status` | Owner | Polling |
| 13 | POST | `/api/fund/expenses` | Admin lớp | Tạo khoản chi |
| 14 | GET | `/api/fund/expenses/{classroomId}` | Member lớp | List chi |
| 15 | POST | `/api/events` | Admin lớp | Tạo sự kiện |
| 16 | GET | `/api/events/{classroomId}` | Member lớp | List sự kiện |
| 17 | POST | `/api/events/{eventId}/volunteer` | Member lớp | Đăng ký |
| 18 | DELETE | `/api/events/{eventId}/volunteer` | Member lớp | Huỷ đăng ký |
| 19 | GET | `/api/events/{eventId}/participants` | Admin lớp | List người đăng ký |
| 20 | PUT | `/api/events/{eventId}/checkin/{userId}` | Admin lớp | Check-in |
| 21 | GET | `/api/events/my/{classroomId}` | Member lớp | List sự kiện đã đăng ký |

**Tổng: 22 endpoint** chia 4 nhóm (Auth 2 / Classroom 3 / Fund 10 / Event 7).

## 7.8. Status code summary

| Code | Số chỗ dùng | Trường hợp |
|---|---|---|
| 200 OK | 19 | Action thành công, trả body |
| 204 No Content | 1 | DELETE /volunteer |
| 400 Bad Request | nhiều | Validate fail, business rule fail |
| 401 Unauthorized | 1 (toàn cục) | Thiếu/sai token |
| 403 Forbidden | nhiều | Không có quyền (authz fail) |
| 500 Internal Server Error | fallback | Lỗi server không lường trước |

## 7.9. Postman Collection

> **TODO:** Export Postman collection `.json` để bàn giao cùng tài liệu. Hiện tại có thể test qua cURL hoặc Postman GUI từng request.

Ví dụ cURL nhanh:
```bash
# 1. Register
curl -X POST http://localhost:8080/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"fullName":"A","email":"a@test.com","password":"12345678"}'

# 2. Login → lấy token
TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"a@test.com","password":"12345678"}' | jq -r .token)

# 3. Tạo lớp
curl -X POST http://localhost:8080/api/classrooms/create \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"className":"K64KTPM3","faculty":"CNTT","academicYear":"K64"}'

# 4. Test 401 (không token)
curl http://localhost:8080/api/classrooms/my
# → {"status":401,"message":"Chưa đăng nhập hoặc token không hợp lệ"}
```
