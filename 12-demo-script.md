# 12 — Demo Script

> Kịch bản demo trước hội đồng. **Tổng thời gian: 8–10 phút.**
> Mục tiêu: thể hiện đủ 3 phân hệ + 5 điểm bảo mật/audit mà hội đồng dễ hỏi.

## Setup trước demo

✅ Trước khi vào phòng demo:
1. MySQL đã chạy, BE đã `./mvnw spring-boot:run`, console không lỗi.
2. 2 thiết bị (laptop + điện thoại) hoặc 2 emulator để mô phỏng Admin + Member.
3. Cấu hình `baseUrl` cho cả 2 thiết bị bằng `--dart-define=API_BASE_URL=...` (xem mục 11.4.2).
4. Đã đăng ký sẵn **3 tài khoản test** (xem `13-tai-khoan-test.md`):
   - `admin@demo.com` — sẽ là Admin.
   - `sv1@demo.com` — Member.
   - `sv2@demo.com` — Member.
5. Postman đã setup environment + collection để demo phần "test 401".
6. **Reset data nếu cần** (xem mục 11.7) — DB sạch để demo từ đầu.

## Kịch bản demo (10 bước)

### Bước 1 — Giới thiệu hệ thống (30 giây)
> "ClassHub là ứng dụng quản lý lớp hành chính, gồm 3 phân hệ chính: **quản lý lớp**, **quỹ lớp** với QR thanh toán, và **sự kiện** với điểm danh. Backend Spring Boot, frontend Flutter, database MySQL."

### Bước 2 — Đăng nhập Admin trên laptop (30 giây)
1. Mở app trên laptop (web hoặc desktop).
2. Đăng nhập `admin@demo.com` / `12345678`.
3. Vào màn Home — đã có sẵn lớp **K64KTPM3** (đã tạo từ trước).

> Nói: "Mã mời được sinh ngẫu nhiên 6 ký tự, duy nhất trên toàn hệ thống."

### Bước 3 — Đăng nhập Member trên điện thoại (30 giây)
1. Mở app trên điện thoại.
2. Đăng nhập `sv1@demo.com` / `12345678`.
3. Đã có sẵn lớp K64KTPM3 (đã join từ trước).

### Bước 4 — Tạo khoản thu (1 phút)
**Trên laptop (Admin):**
1. Tap card lớp K64KTPM3 → vào ClassroomDetail.
2. Sang tab "Khoản thu" → bấm FAB "Tạo đợt thu".
3. Nhập: title = "Quỹ lớp tháng 5", amount = 50000, deadline = ngày mai → bấm Tạo.

> Nói: "**Khi admin tạo, hệ thống tự sinh bản ghi nợ cho cả 3 thành viên cùng lúc** — không phải tạo thủ công cho từng người."

### Bước 5 — Sinh viên xem khoản nợ + lấy QR (1.5 phút)
**Trên điện thoại (Member):**
1. Mở lớp K64KTPM3 → tab Khoản thu.
2. Section "Khoản của bạn" hiển thị: **"Quỹ lớp tháng 5 — 50.000 đ — Hạn dd/mm/yyyy"**.

> Nói: "Sinh viên thấy ngay số tiền và hạn — không cần hỏi admin."

3. Bấm "Xem QR" → màn QR mở.
4. QR VietQR hiển thị, kèm số tiền + **nội dung chuyển khoản duy nhất** (`QUY...-SV...-timestamp`).

> Nói: "Nội dung chuyển khoản là **mã duy nhất**, admin đối chiếu sao kê là biết em nào đã chuyển."

5. Bấm copy nội dung CK (đã copy vào clipboard).
6. Phía dưới hiển thị: "Đang chờ admin xác nhận... (tự cập nhật)" với spinner.

> Nói: "App polling trạng thái mỗi 5 giây, tự dừng khi admin xác nhận."

### Bước 6 — Admin xác nhận thanh toán (1 phút)
**Trên laptop (Admin):**
1. Tap card đợt thu "Quỹ lớp tháng 5" → vào danh sách payments.
2. Thấy sv1 (chưa đóng — chip cam) và sv2 (chưa đóng).
3. Bấm "Xác nhận" cạnh sv1 → confirm dialog hiện ra.

> Nói: "Có **confirm dialog** để chống lỡ tay — vì backend chặn xác nhận 2 lần (idempotent)."

4. Bấm "Xác nhận" lần 2 → snackbar xanh "Đã xác nhận thanh toán".
5. sv1 chuyển sang chip xanh "Đã xác nhận bởi Admin Demo".

> Nói: "Backend lưu **ai đã xác nhận** và **lúc nào** — đây là audit trail."

### Bước 7 — Member thấy trạng thái cập nhật (real-time qua polling) (30 giây)
**Trên điện thoại (Member):**
- Vẫn ở màn QR.
- Trong khoảng **≤ 5 giây**, status đổi từ orange "Đang chờ" → xanh "**Đã thanh toán & được xác nhận**".

> Nói: "Sinh viên thấy trạng thái cập nhật mà không cần refresh — đây là cảm giác 'real-time' nhờ polling."

### Bước 8 — Demo phân hệ Sự kiện (1.5 phút)

**Trên laptop (Admin):**
1. Sang tab "Sự kiện" → FAB "Tạo sự kiện".
2. Tạo "Họp lớp cuối kỳ", location = "P.A201", chọn ngày giờ.

**Trên điện thoại (Member):**
3. Sang tab "Sự kiện" → kéo refresh.
4. Thấy sự kiện mới. Bấm "Đăng ký" → chip "Đã đăng ký" hiện ra.

**Trên laptop (Admin):**
5. Vào sự kiện → "Người tham gia" → thấy sv1.
6. Bấm "Check-in" → confirm dialog → check-in.

> Nói: "Tương tự xác nhận thanh toán, **lưu ai check-in và lúc nào** để truy vết."

7. Counter "Check-in: 1/1" cập nhật.

### Bước 9 — Demo bảo mật (2 phút) — **Quan trọng nhất**

Mở Postman / cURL. Chứng minh 3 điểm bảo mật:

**9a. Token sai → 401 JSON (không phải HTML login form)**
```bash
curl -i http://localhost:8080/api/classrooms/my
```
→ Console hiển thị:
```
HTTP/1.1 401
{"status":401,"message":"Chưa đăng nhập hoặc token không hợp lệ"}
```

> "API trả JSON gọn gàng — không lộ stacktrace, không trả HTML login mặc định của Spring."

**9b. Member tạo khoản thu → 403**
```bash
# Login với sv1
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -d '{"email":"sv1@demo.com","password":"12345678"}' | jq -r .token)

# Sv1 (Member) cố tạo khoản thu
curl -i -X POST http://localhost:8080/api/fund/collections \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"classroomId":1,"title":"Test","amount":1000}'
```
→
```
HTTP/1.1 403
{"status":403,"message":"Chỉ Ban cán sự được phép thao tác"}
```

> "Đây là **authorization theo lớp** — không chỉ check authenticated, mà còn check role của user trong lớp đó."

**9c. Cross-class attack → 403** (nếu có thời gian)
- Tạo lớp K thứ 2 với tài khoản khác.
- Sv1 (admin lớp 1) cố confirm payment lớp 2:
```bash
curl -i -X PUT http://localhost:8080/api/fund/payments/{paymentIdOfClass2}/confirm \
  -H "Authorization: Bearer $TOKEN_SV1"
```
→ 403 "Bạn không thuộc lớp này"

> "Admin lớp này **không thao tác được** dữ liệu lớp khác."

### Bước 10 — Tổng kết (30 giây)

> "Đã demo:
> 1. **3 phân hệ** (Auth, Quỹ, Sự kiện) chạy end-to-end.
> 2. **Bảo mật chặt**: JWT validate, authorization theo lớp, audit trail.
> 3. **Trải nghiệm**: QR + polling, real-time cảm giác, confirm dialog chống lỡ tay.
>
> Hệ thống hiện đã đủ điều kiện sử dụng thực tế cho 1 lớp đại học."

---

## Bonus — Câu hỏi hội đồng dễ hỏi + Đáp án sẵn

| Câu hỏi | Đáp án |
|---|---|
| "Khác Excel/Zalo thế nào?" | "Dữ liệu tập trung, phân quyền theo lớp, sinh viên tự xem nợ, audit trail." |
| "QR có tự động không?" | "QR sinh URL VietQR, sinh viên CK, admin đối chiếu sao kê + bấm xác nhận. Đây là **bán tự động**. Tự động hoàn toàn cần webhook ngân hàng — hướng phát triển." |
| "Token lưu ở đâu?" | "Server stateless, không lưu. Client lưu SharedPreferences, mỗi request gắn Authorization Bearer." |
| "Mã mời sinh thế nào?" | "6 ký tự uppercase từ UUID, unique constraint ở DB." |
| "Thầy có token, có vào được API người khác không?" | "Token có claim userId. Service luôn check `userId == payment.user.id` cho action lấy QR/status. Còn action chỉnh sửa thì check role qua `ClassMember.role`." |
| "Sinh viên join lớp sau khi đã có khoản thu thì sao?" | "Có. `joinClassroom` query mọi collection của lớp + sinh payment bổ sung cho user mới. Đã test." |
| "Confirm thanh toán 2 lần thì sao?" | "Backend chặn idempotent — trả 400 'đã được xác nhận'. Frontend cũng có confirm dialog." |
| "Có test case chưa?" | "Có 28 test case đặc tả ở `10-kiem-thu.md`, đã chạy manual qua Postman. Test tự động JUnit + MockMvc đưa vào giai đoạn tiếp theo." |
| "Hạn chế hiện tại?" | "(1) Chưa có tab Thành viên — chờ API BE. (2) Chưa có dashboard thống kê quỹ. (3) Backend hiện vẫn 2 cấp ADMIN/MEMBER; frontend đã gom helper `UserRoles.isAdminLike` để sẵn sàng coi OWNER như quyền admin nếu BE mở rộng. (4) Đối soát ngân hàng thủ công. Tất cả ở mục Hướng phát triển." |

## Lỗi có thể gặp khi demo + Cách xử lý

| Tình huống | Xử lý |
|---|---|
| BE đột nhiên 500 | Có thể DB mất kết nối → restart MySQL → restart BE. Hoặc giải thích "kết nối mạng nội bộ, mình sẽ chuyển sang demo bằng video đã quay sẵn." |
| Polling không cập nhật | Có thể mất mạng tạm thời — refresh thủ công sẽ thấy. Hoặc giải thích "polling 5s nên đôi khi chậm hơn 1 nhịp." |
| QR không tải được | Vẫn còn payment code copy được. Giải thích "URL VietQR cần internet — đây là dịch vụ ngoài." |
| App crash | Bình tĩnh, mở lại app, login lại. Token vẫn còn trong SharedPreferences. |

## Backup plan: Demo bằng video

Quay sẵn 1 video 3-5 phút chạy đủ 10 bước trên (dùng `flutter screenshot` + record màn hình). Để dự phòng nếu môi trường demo có vấn đề.
