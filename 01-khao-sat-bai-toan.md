# 01 — Khảo sát bài toán

## 1.1. Bối cảnh

Sinh viên đại học sinh hoạt theo **lớp hành chính** (vd K64 KTPM3). Mỗi lớp có:
- Một ban cán sự (lớp trưởng, lớp phó, thủ quỹ).
- 30–60 sinh viên thành viên.
- Hoạt động định kỳ: thu quỹ lớp, tổ chức sự kiện, thông báo điểm danh.

Quy mô nhỏ nhưng tần suất công việc cao: trung bình mỗi tháng có 1–2 đợt thu quỹ và 1–2 sự kiện cần điểm danh.

## 1.2. Hiện trạng — Lớp đang quản lý như thế nào?

Khảo sát thực tế (qua phỏng vấn 5 ban cán sự lớp K63–K65 Khoa CNTT) cho thấy:

| Công việc | Công cụ đang dùng |
|---|---|
| Thông báo khoản thu | Zalo / Messenger group |
| Theo dõi ai đã đóng | Excel / Google Sheets (thủ công) |
| Sinh viên xác nhận đã đóng | Tin nhắn riêng cho ban cán sự |
| Đăng ký tham gia sự kiện | Google Forms |
| Điểm danh tham gia | Gọi tên trực tiếp / chụp ảnh điểm danh |
| Báo cáo thu chi cuối kỳ | Excel tổng kết |

## 1.3. Vấn đề rút ra từ khảo sát

### 1.3.1. Dữ liệu phân tán
- Khoản thu nằm trong tin nhắn Zalo.
- Trạng thái đóng tiền nằm trong Excel.
- Đăng ký sự kiện nằm trong Google Forms.
- Lịch sử các kỳ trước nằm trong file cũ ai giữ thì người đó có.

→ Khi cần tra cứu "tháng 3 năm ngoái lớp đã thu bao nhiêu" thì không có cách trả lời nhanh.

### 1.3.2. Khó phân quyền
- Excel/Sheets chia sẻ cho cả lớp → ai cũng có thể sửa số liệu, dễ nhầm/cố ý chỉnh.
- Nếu chỉ chia sẻ riêng cho ban cán sự → sinh viên không tự xem được nợ của mình.

### 1.3.3. Sinh viên khó tự tra cứu nợ
- Phải nhắn ban cán sự để hỏi "em còn nợ khoản nào không?".
- Ban cán sự phải mở Excel, tìm tên → tốn thời gian cho cả 2 bên.

### 1.3.4. Ban cán sự mất thời gian tổng hợp
- Cuối mỗi đợt thu, lớp trưởng phải:
  - Xem từng tin nhắn xác nhận trong group.
  - Mở Excel, tick từng người.
  - Cộng tay tổng số tiền đã thu.
- Trung bình tốn 30–60 phút mỗi đợt.

### 1.3.5. Khó tra cứu lịch sử
- File Excel của kỳ trước có khi đã mất hoặc người giữ file đã ra trường.
- Không có hệ thống lưu trữ tập trung.

### 1.3.6. Khó thống kê sự kiện
- Đăng ký Google Forms, điểm danh giấy → không gắn 2 dữ liệu với nhau được.
- Không trả lời được "em A trong cả năm tham gia mấy sự kiện".

### 1.3.7. Dễ nhầm lẫn khi xác nhận thủ công
- Tin nhắn "em đóng rồi ạ" lẫn trong group hàng trăm tin nhắn/ngày.
- Quên cập nhật vào Excel → đến kỳ sau lại đòi tiền nhầm.

## 1.4. Nhu cầu của các đối tượng

### 1.4.1. Ban cán sự (sẽ là Admin trong hệ thống)
- Cần tạo khoản thu một lần, hệ thống tự sinh nợ cho từng sinh viên.
- Cần danh sách rõ ai đóng, ai chưa, kèm thời gian.
- Cần một cú click để xác nhận thanh toán.
- Cần tổng hợp tự động: tổng thu, tổng chi, số dư quỹ.
- Cần tạo sự kiện + điểm danh trên cùng một hệ thống.

### 1.4.2. Sinh viên (Member)
- Cần tự xem mình đang nợ khoản nào, hạn đóng khi nào.
- Cần thanh toán tiện lợi (QR ngân hàng) thay vì chuyển tay/tiền mặt.
- Cần thấy ngay trạng thái sau khi đóng tiền.
- Cần xem lịch sử các kỳ trước.
- Cần biết các sự kiện sắp tới + đăng ký tham gia.

### 1.4.3. Lớp (về mặt vận hành)
- Dữ liệu phải lưu trữ tập trung, không phụ thuộc người giữ file.
- Bàn giao giữa các khoá ban cán sự phải nhẹ nhàng (kế nhiệm chỉ cần được cấp quyền Admin).

## 1.5. Yêu cầu sản phẩm rút ra

Từ phân tích trên, hệ thống ClassHub cần đáp ứng:

| Yêu cầu | Mô tả |
|---|---|
| Quản lý lớp tập trung | Mọi dữ liệu trong 1 hệ thống, truy cập mọi lúc |
| Phân quyền theo lớp | Member chỉ thấy lớp mình, Admin chỉ quản lý lớp mình có quyền |
| Quỹ lớp đầy đủ | Tạo khoản thu, xem nợ, xác nhận thanh toán, lịch sử |
| Hỗ trợ QR thanh toán | Sinh QR VietQR; xác nhận bán tự động bởi admin |
| Sự kiện + điểm danh | Tạo sự kiện, sinh viên đăng ký, admin điểm danh có/không có mặt |
| Bảo mật cơ bản | Mật khẩu mã hoá, xác thực JWT, không lộ dữ liệu lớp khác |
| Đa nền tảng | Ứng dụng di động (Flutter) — sinh viên ai cũng có smartphone |

## 1.6. Phạm vi không làm (giới hạn)

Để tập trung làm đúng phần lõi, ClassHub **KHÔNG** xử lý các phần sau (đưa vào hướng phát triển):
- Đối soát tự động với ngân hàng (cần API/webhook VietQR + hợp đồng).
- Push notification thật (Firebase Cloud Messaging) — chỉ làm thông báo trong app.
- Hỗ trợ iOS chính thức — tập trung Android trước.
- Tích hợp hệ thống của nhà trường (LDAP/SSO).
- Xuất báo cáo PDF/Excel — sẽ dùng thống kê trong app.

## 1.7. Kết luận khảo sát

Hệ thống ClassHub giải quyết **4 vấn đề chính** mà các công cụ rời rạc hiện tại (Zalo + Excel + Google Forms) không giải quyết được:

1. **Tập trung hoá** dữ liệu lớp.
2. **Phân quyền** rõ ràng giữa Admin và Member.
3. **Tự động hoá** việc sinh nợ và tổng hợp thống kê.
4. **Trải nghiệm sinh viên** tự phục vụ (xem nợ, lấy QR, đăng ký sự kiện).
