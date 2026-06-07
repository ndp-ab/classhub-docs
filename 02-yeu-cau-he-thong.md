# 02 — Yêu cầu hệ thống

## 2.1. Yêu cầu chức năng (Functional Requirements)

### FR-1. Quản lý tài khoản
| Mã | Chức năng | Actor |
|---|---|---|
| FR-1.1 | Đăng ký tài khoản bằng email + mật khẩu | Guest |
| FR-1.2 | Đăng nhập, hệ thống trả JWT | Guest |
| FR-1.3 | Đăng xuất, xoá token khỏi thiết bị | Member/Admin |

### FR-2. Quản lý lớp
| Mã | Chức năng | Actor |
|---|---|---|
| FR-2.1 | Tạo lớp mới — người tạo tự động là Admin | Member/Admin |
| FR-2.2 | Lớp sinh mã mời 6 ký tự duy nhất | Hệ thống |
| FR-2.3 | Tham gia lớp bằng mã mời | Member |
| FR-2.4 | Xem danh sách lớp đã tham gia | Member/Admin |
| FR-2.5 | Xem chi tiết lớp (tên, khoa, khóa, mã mời, vai trò) | Member/Admin |

### FR-3. Quản lý quỹ lớp — Khoản thu
| Mã | Chức năng | Actor |
|---|---|---|
| FR-3.1 | Tạo đợt thu (title, amount, deadline) | Admin |
| FR-3.2 | Khi tạo đợt thu, hệ thống tự sinh bản ghi nợ cho TẤT CẢ thành viên | Hệ thống |
| FR-3.3 | Sinh viên join lớp SAU khi đã có đợt thu vẫn được tự sinh bản ghi nợ | Hệ thống |
| FR-3.4 | Xem danh sách đợt thu của lớp | Member của lớp |
| FR-3.5 | Xem ai đã đóng / chưa đóng của 1 đợt thu | Admin của lớp |
| FR-3.6 | Xác nhận một sinh viên đã đóng tiền | Admin của lớp |
| FR-3.7 | Hệ thống lưu **ai xác nhận** và **lúc nào** | Hệ thống |
| FR-3.8 | Chặn xác nhận lại 1 khoản đã được xác nhận | Hệ thống |
| FR-3.9 | Sinh viên xem nợ cá nhân của mình trong 1 lớp | Member |
| FR-3.10 | Sinh viên lấy QR thanh toán VietQR cho khoản nợ của mình | Member |
| FR-3.11 | Sinh viên polling trạng thái khoản nợ (PENDING → CONFIRMED) | Member |

### FR-4. Quản lý quỹ lớp — Khoản chi
| Mã | Chức năng | Actor |
|---|---|---|
| FR-4.1 | Tạo khoản chi (title, amount, reason) | Admin |
| FR-4.2 | Xem danh sách khoản chi của lớp | Member của lớp |

### FR-5. Quản lý sự kiện
| Mã | Chức năng | Actor |
|---|---|---|
| FR-5.1 | Tạo sự kiện (title, description, location, eventTime) | Admin |
| FR-5.2 | Xem danh sách sự kiện của lớp | Member của lớp |
| FR-5.3 | Sinh viên xung phong tham gia | Member |
| FR-5.4 | Hệ thống chặn đăng ký trùng | Hệ thống |
| FR-5.5 | Sinh viên huỷ đăng ký nếu chưa check-in | Member |
| FR-5.6 | Hệ thống chặn huỷ sau khi đã check-in | Hệ thống |
| FR-5.7 | Xem danh sách người đăng ký của 1 sự kiện | Admin của lớp |
| FR-5.8 | Check-in (đánh dấu có mặt) cho từng sinh viên | Admin của lớp |
| FR-5.9 | Hệ thống lưu **ai check-in** và **lúc nào** | Hệ thống |
| FR-5.10 | Sinh viên xem các sự kiện mình đã đăng ký | Member |
| FR-5.11 | Member gửi ảnh minh chứng điểm danh cho sự kiện đã đăng ký (upload multipart/form-data) | Member |
| FR-5.12 | Hệ thống lưu file ảnh ngoài DB, lưu metadata và đường dẫn trong DB | Hệ thống |
| FR-5.13 | Admin xem ảnh minh chứng của từng người tham gia | Admin của lớp |
| FR-5.14 | Admin duyệt ảnh minh chứng để xác nhận check-in | Admin của lớp |
| FR-5.15 | Admin từ chối ảnh minh chứng kèm lý do | Admin của lớp |
| FR-5.16 | Member được gửi lại ảnh nếu submission bị từ chối | Member |

## 2.2. Yêu cầu phi chức năng (Non-functional Requirements)

### NFR-1. Bảo mật
| Mã | Yêu cầu | Trạng thái thực hiện |
|---|---|---|
| NFR-1.1 | Mật khẩu mã hoá bằng BCrypt trước khi lưu DB | ✅ |
| NFR-1.2 | Authentication bằng JWT (HS256, hạn 24h) | ✅ |
| NFR-1.3 | Mọi API trừ /api/auth/** yêu cầu Bearer token | ✅ |
| NFR-1.4 | API trả 401 JSON khi token thiếu/sai (không trả HTML login form) | ✅ |
| NFR-1.5 | Authorization theo lớp: user chỉ truy cập dữ liệu lớp mình | ✅ |
| NFR-1.6 | Member không gọi được API của Admin → trả 403 | ✅ |
| NFR-1.7 | DTO không chứa password hash trong response | ✅ |
| NFR-1.8 | Audit trail: lưu ai xác nhận thanh toán, ai check-in | ✅ |

### NFR-2. Validation
| Mã | Yêu cầu | Trạng thái |
|---|---|---|
| NFR-2.1 | Email phải đúng format | ✅ `@Email` |
| NFR-2.2 | Email không trùng | ✅ unique constraint |
| NFR-2.3 | Mật khẩu không rỗng | ✅ `@NotBlank` |
| NFR-2.4 | Số tiền khoản thu/chi phải > 0 | ✅ `@DecimalMin("0.01")` |
| NFR-2.5 | Tiêu đề khoản thu/chi/sự kiện không rỗng | ✅ `@NotBlank` |
| NFR-2.6 | Mã mời lớp phải tồn tại khi join | ✅ check ở service |

### NFR-3. Trải nghiệm
| Mã | Yêu cầu |
|---|---|
| NFR-3.1 | Ứng dụng chạy trên Android (emulator + thiết bị thật) |
| NFR-3.2 | UI tiếng Việt toàn bộ |
| NFR-3.3 | Confirm dialog trước action không hoàn tác (xác nhận thanh toán, check-in) |
| NFR-3.4 | Pull-to-refresh trên danh sách |
| NFR-3.5 | Loading / error / empty state rõ ràng cho từng màn |

### NFR-6. Upload ảnh minh chứng
| Mã | Yêu cầu | Trạng thái |
|---|---|---|
| NFR-6.1 | Ảnh upload tối đa 5MB | ✅ Validate ở FileStorageService |
| NFR-6.2 | Chỉ chấp nhận định dạng jpg/jpeg/png | ✅ ALLOWED_EXTENSIONS |
| NFR-6.3 | Ảnh không lưu binary/base64 trong DB — chỉ lưu đường dẫn | ✅ Thiết kế tách bảng |
| NFR-6.4 | `/uploads/**` public cho MVP/demo để Flutter hiển thị ảnh nhanh | ✅ WebMvcConfig |
| NFR-6.5 | Hướng phát triển: API download có JWT/authorization khi cần bảo mật cao hơn | 🟢 Hướng phát triển |

### NFR-4. Hiệu năng
| Mã | Yêu cầu |
|---|---|
| NFR-4.1 | API trả về < 1s với DB local cùng máy |
| NFR-4.2 | Polling QR status mỗi 5s, dừng polling khi đã được xác nhận |
| NFR-4.3 | Dispose timer khi thoát màn QR (chống rò rỉ tài nguyên) |

### NFR-5. Dữ liệu
| Mã | Yêu cầu |
|---|---|
| NFR-5.1 | Số tiền dùng `BigDecimal` (không dùng `double` để tránh lỗi làm tròn) |
| NFR-5.2 | Timestamp dùng `LocalDateTime` server timezone (Asia/Ho_Chi_Minh) |
| NFR-5.3 | Mọi entity có `createdAt` (auto `@CreationTimestamp`) |
| NFR-5.4 | Khoá ngoại dùng `@ManyToOne(fetch=LAZY)` (tránh load không cần thiết) |
| NFR-5.5 | Unique constraint chống join lớp trùng / đăng ký sự kiện trùng |

## 2.3. Giới hạn phạm vi (Out of scope)

| # | Tính năng | Lý do không làm |
|---|---|---|
| OS-1 | Push notification (Firebase) | Cần FCM project + tích hợp tốn thời gian |
| OS-2 | Đối soát tự động với ngân hàng | Cần hợp đồng với bank để dùng API webhook |
| OS-3 | Hỗ trợ iOS | Tập trung Android trước; iOS cần Mac + Apple Developer account |
| OS-4 | Tích hợp hệ thống nhà trường (LDAP/SSO) | Vượt phạm vi đồ án sinh viên |
| OS-5 | Refresh token | JWT 24h đủ cho demo; thêm refresh token làm UI phức tạp |
| OS-6 | Import/export Excel | Hệ thống đã thay thế Excel, không cần xuất Excel lại |
| OS-7 | Chat realtime trong lớp | Đã có Zalo, không trùng nhu cầu |
| OS-8 | OCR sao kê ngân hàng | Sản phẩm thương mại sau này |

→ Toàn bộ OS-1 đến OS-8 được trình bày trong mục **Hướng phát triển** của báo cáo.

## 2.4. Mức độ hoàn thành so với yêu cầu

| Nhóm | Hoàn thành | Ghi chú |
|---|---|---|
| FR-1 Tài khoản | 3/3 ✅ | |
| FR-2 Lớp | 5/5 ✅ | |
| FR-3 Khoản thu | 11/11 ✅ | |
| FR-4 Khoản chi | 2/2 ✅ | |
| FR-5 Sự kiện | 16/16 ✅ code / ⚠️ cần test device | FR-5.11–5.16 đã implement BE+FE, build/analyze pass; cần kiểm thử thực tế trên Android device cho luồng chụp ảnh → upload → duyệt |
| NFR-1 Bảo mật | 8/8 ✅ | |
| NFR-2 Validation | 6/6 ✅ | |
| NFR-3 Trải nghiệm | 5/5 ✅ | |
| NFR-4 Hiệu năng | 3/3 ✅ | |
| NFR-5 Dữ liệu | 5/5 ✅ | |
| NFR-6 Upload ảnh | 4/5 ✅ / 1🟢 | NFR-6.5 là hướng phát triển |

**Tổng: 64 yêu cầu** (thêm 6 FR camera check-in + 5 NFR upload ảnh). Code đã implement toàn bộ; FR-5.11–5.16 và NFR-6 cần kiểm thử E2E trên thiết bị thật để xác nhận hoàn chỉnh.
