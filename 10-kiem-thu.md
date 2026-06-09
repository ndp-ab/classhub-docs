# 10 — Kiểm thử

## 10.1. Phạm vi kiểm thử

| Loại test | Trạng thái | Ghi chú |
|---|---|---|
| Manual test (Postman + Flutter) | ✅ Đã làm theo demo script | Xem `12-demo-script.md` |
| Test case TC01–TC53 đặc tả | ✅ Đã liệt kê dưới | |
| Test case tự động (JUnit + MockMvc) | 🟡 Chưa implement | Sẽ làm trong giai đoạn tiếp theo |
| Test trên thiết bị thật | ⚠️ Cần test trước demo | Chạy FE với `--dart-define=API_BASE_URL=http://<IP-LAN>:8080/api` |

## 10.2. Bảng test case TC01–TC53

Test case bao phủ 7 nhóm: Auth, Classroom, Classroom Bank Account, Fund, Event, Camera Check-in, Notification.

| Mã TC | Module | Chức năng | Điều kiện | Kết quả mong đợi | Status |
|---|---|---|---|---|---|
| TC01 | Auth | Đăng ký tài khoản mới | Email chưa tồn tại, password ≥ 8 ký tự | 200, trả JWT + userId | ⬜ |
| TC02 | Auth | Đăng ký với email đã tồn tại | Email đã có trong DB | 400 "Email đã được sử dụng" | ⬜ |
| TC03 | Auth | Đăng ký với email sai format | Email = "abc" | 400 + field error "email" | ⬜ |
| TC04 | Auth | Đăng nhập đúng | Email + password đúng | 200, trả JWT mới | ⬜ |
| TC05 | Auth | Đăng nhập sai password | Sai password | 400 "Mật khẩu không đúng" | ⬜ |
| TC06 | Auth | Gọi API thiếu token | `GET /classrooms/my` không kèm Authorization | 401 JSON `{status:401, message:...}` | ⬜ |
| TC07 | Auth | Gọi API với token sai | Authorization Bearer "abc.def.ghi" | 401 | ⬜ |
| TC08 | Classroom | Tạo lớp | User đã đăng nhập | 200, sinh inviteCode 6 ký tự, user tự động Admin | ⬜ |
| TC09 | Classroom | Join lớp với mã hợp lệ | User chưa ở lớp đó | 200, role=MEMBER | ⬜ |
| TC10 | Classroom | Join lớp đã tham gia rồi | User đã ở lớp | 400 "Bạn đã tham gia lớp này rồi" | ⬜ |
| TC11 | Classroom | Join với mã không tồn tại | inviteCode = "ZZZZZZ" | 400 "Mã tham gia không hợp lệ" | ⬜ |
| TC11a | Bank Account | Admin tạo/cập nhật tài khoản ngân hàng | Admin lớp, data hợp lệ | 200, tạo bản mới active=true, cũ chuyển inactive | ⬜ |
| TC11b | Bank Account | Member xem tài khoản ngân hàng | Member lớp | 200, trả bank account active | ⬜ |
| TC11c | Bank Account | Member cố cập nhật tài khoản | Member không phải Admin | **403** "Bạn không phải Admin của lớp" | ⬜ |
| TC11d | Bank Account | User ngoài lớp xem tài khoản | User không thuộc lớp | **403** "Bạn không thuộc lớp này" | ⬜ |
| TC11e | Bank Account | Admin xem lịch sử tài khoản | Admin lớp | 200, trả danh sách tất cả bản ghi (active + inactive) | ⬜ |
| TC11f | Bank Account | Tạo khoản thu khi lớp chưa có tài khoản | Lớp chưa có bank account active | **400** "Vui lòng cấu hình tài khoản ngân hàng nhận tiền trước khi tạo khoản thu" | ⬜ |
| TC11g | Bank Account | QR lấy tài khoản từ DB | Lớp có bank account active, member lấy QR | 200, QR chứa bankName/accountNo/accountName từ DB, không dùng config cố định | ⬜ |
| TC12 | Fund | Member tạo khoản thu | User role=MEMBER | **403** "Chỉ Ban cán sự được phép thao tác" | ⬜ |
| TC13 | Fund | Admin tạo khoản thu hợp lệ | amount=50000, deadline=tương lai | 200, sinh N payment cho N member | ⬜ |
| TC14 | Fund | Tạo khoản thu amount=0 | amount = 0 | **400** "Số tiền phải lớn hơn 0" | ⬜ |
| TC15 | Fund | Tạo khoản thu amount âm | amount = -1000 | 400 validation fail | ⬜ |
| TC16 | Fund | Admin xác nhận thanh toán | Payment chưa xác nhận | 200, set `paidAt` + `confirmedBy` | ⬜ |
| TC17 | Fund | Admin xác nhận lại payment đã confirmed | Đã confirm 1 lần | **400** "Khoản thu này đã được xác nhận" (idempotency) | ⬜ |
| TC18 | Fund | Admin lớp A xác nhận payment lớp B | Cross-class attack | **403** "Bạn không thuộc lớp này" | ⬜ |
| TC19 | Fund | Member xem QR payment của người khác | userId != payment.user.id | **403** "Bạn chỉ có thể xem QR của chính mình" | ⬜ |
| TC20 | Fund | Member join lớp đã có khoản thu | Lớp có 2 collection, user join sau | Auto sinh 2 payment cho user mới (B6) | ⬜ |
| TC21 | Event | Admin tạo sự kiện | eventTime hợp lệ | 200 | ⬜ |
| TC22 | Event | Member đăng ký tham gia | Chưa đăng ký lần nào | 200, tạo EventParticipant | ⬜ |
| TC23 | Event | Đăng ký trùng | Đã đăng ký rồi | 400 "Bạn đã đăng ký..." | ⬜ |
| TC24 | Event | User ngoài lớp đăng ký | userId không trong class_members | 403 "Bạn không thuộc lớp này" | ⬜ |
| TC25 | Event | Member huỷ đăng ký chưa check-in | checkedIn=false | 204 | ⬜ |
| TC26 | Event | Member huỷ sau khi đã check-in | checkedIn=true | 400 "Không thể hủy đăng ký sau khi đã check-in" | ⬜ |
| TC27 | Event | Admin check-in thủ công | Participant chưa check-in | 200, set `checkedBy` + `checkedInAt` | ⬜ |
| TC28 | Event | Check-in lại sinh viên đã check-in | checkedIn=true | 400 "Sinh viên này đã được check-in rồi" | ⬜ |
| TC-FILE-01 | Camera Check-in | Member gửi ảnh điểm danh hợp lệ | Ảnh jpg, size < 5MB, đã đăng ký sự kiện | 200, tạo submission PENDING, lưu file vào `classhub-uploads/` | ⬜ |
| TC-FILE-02 | Camera Check-in | Gửi file không phải ảnh (pdf) | contentType = application/pdf hoặc extension .pdf | 400 "File phải là ảnh" | ⬜ |
| TC-FILE-03 | Camera Check-in | Admin duyệt ảnh PENDING | submission.status=PENDING | 200, set APPROVED, set EventParticipant.checkedIn=true | ⬜ |
| TC-FILE-04 | Camera Check-in | Admin từ chối ảnh kèm lý do | submission.status=PENDING | 200, set REJECTED, lưu rejectedReason | ⬜ |
| TC-FILE-05 | Camera Check-in | Member gửi lại ảnh sau khi bị từ chối | submission REJECTED, member gửi mới | 200, tạo submission PENDING mới | ⬜ |
| TC-NOTI-01 | Notification | Lấy danh sách thông báo | User đã đăng nhập, có notification recipient | `GET /api/notifications?page=0&size=20` trả `Page` có `content`, chỉ notification của user hiện tại | ⬜ |
| TC-NOTI-02 | Notification | Đếm unread count | User có N recipient `is_read=false` | `GET /api/notifications/unread-count` trả `{ "count": N }` | ⬜ |
| TC-NOTI-03 | Notification | Mark one as read | Recipient thuộc user hiện tại, `is_read=false` | `PUT /api/notifications/{recipientId}/read` trả `isRead=true`, DB set `read_at` | ⬜ |
| TC-NOTI-04 | Notification | Mark all as read | User có nhiều notification chưa đọc | `PUT /api/notifications/read-all` trả `{ "count": 0 }`, tất cả recipient của user được set read | ⬜ |
| TC-NOTI-05 | Notification | Admin tạo event → member nhận notification | Admin lớp tạo sự kiện mới | Member trong lớp nhận `EVENT_CREATED`, `targetType=EVENT`, `targetId=event.id` | ⬜ |
| TC-NOTI-06 | Notification | Admin tạo khoản thu → member nhận notification | Admin lớp tạo khoản thu mới | Member trong lớp nhận `COLLECTION_CREATED`, `targetType=FUND_COLLECTION`, `targetId=collection.id` | ⬜ |
| TC-NOTI-07 | Notification | Người tạo không nhận notification của chính mình | Admin tạo event/khoản thu | Không tạo `NotificationRecipient` cho user tạo nghiệp vụ | ⬜ |
| TC-NOTI-08 | Notification Security | User không đọc/sửa recipient của user khác | User A gọi `PUT /api/notifications/{recipientIdOfUserB}/read` | 403, recipient của user B không đổi trạng thái | ⬜ |
| TC-NOTI-09 | FE Notification | Home badge hiển thị đúng | User có unread count > 0 | `HomeScreen` hiển thị badge số chưa đọc trên icon chuông; quay lại từ `NotificationScreen` thì refresh count | ⬜ |
| TC-NOTI-10 | FE Notification | NotificationScreen hiển thị list/empty/error/loading | Mock/API trả đủ 4 trạng thái | UI dùng loading, empty state, error state và list item đúng dữ liệu | ⬜ |
| TC-NOTI-11 | FE Notification | Pull refresh | Đang ở `NotificationScreen` | Kéo refresh gọi lại `GET /api/notifications`, list cập nhật | ⬜ |
| TC-NOTI-12 | FE Notification | Tap item mark read | Item đang unread | Tap item gọi mark read, dot unread biến mất, không điều hướng sâu | ⬜ |
| TC-NOTI-13 | FE Notification | Mark all read | Có ít nhất 1 item unread | Bấm "Đánh dấu tất cả đã đọc" gọi API, toàn bộ item chuyển sang read | ⬜ |

**Tổng: 53 test case** (thêm 5 test case cho Camera Check-in và 13 test case cho Notification MVP).

## 10.3. Bộ dữ liệu test

### Users
| ID | Email | Password | Vai trò trong test |
|---|---|---|---|
| 1 | admin1@test.com | 12345678 | Admin lớp K64KTPM3 |
| 2 | admin2@test.com | 12345678 | Admin lớp K64KHMT |
| 3 | member1@test.com | 12345678 | Member lớp K64KTPM3 |
| 4 | member2@test.com | 12345678 | Member lớp K64KTPM3 |
| 5 | outsider@test.com | 12345678 | Không tham gia lớp nào |

### Classes
- K64KTPM3 — Admin: user 1. Members: user 3, 4.
- K64KHMT — Admin: user 2. Không có member.

### Collections
- Collection #1 (K64KTPM3): "Quỹ tháng 5", 50.000đ, deadline 31/5.

## 10.4. Hướng dẫn chạy test thủ công

### 10.4.1. Bằng Postman

**Setup:**
1. Tạo collection Postman "ClassHub".
2. Tạo environment variable `baseUrl = http://localhost:8080/api` và `token` (rỗng ban đầu).

**Script tests trên Postman (sau request login):**
```javascript
// Tab Tests của request Login
var json = pm.response.json();
pm.environment.set("token", json.token);
pm.environment.set("userId", json.userId);
```

**Mỗi request authenticated:**
- Tab Authorization → Type "Bearer Token" → token = `{{token}}`.

**Chạy các test case** theo bảng trên, ghi kết quả Pass/Fail vào cột Status.

### 10.4.2. Bằng cURL

```bash
# Login lấy token
TOKEN=$(curl -s -X POST $BASE/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"admin1@test.com","password":"12345678"}' | jq -r .token)

# TC06: thiếu token → expect 401
curl -i $BASE/classrooms/my
# HTTP/1.1 401 ✅
# {"status":401,"message":"Chưa đăng nhập hoặc token không hợp lệ"} ✅

# TC12: Member tạo khoản thu → expect 403
MEMBER_TOKEN=$(curl -s -X POST $BASE/auth/login \
  -d '{"email":"member1@test.com","password":"12345678"}' | jq -r .token)
curl -i -X POST $BASE/fund/collections \
  -H "Authorization: Bearer $MEMBER_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"classroomId":1,"title":"Test","amount":1000}'
# HTTP/1.1 403 ✅
# {"status":403,"message":"Chỉ Ban cán sự được phép thao tác"} ✅
```

## 10.5. Hướng dẫn implement test tự động (giai đoạn tiếp theo)

### Spring Boot Test với MockMvc

Mẫu cho TC12 (Member tạo khoản thu → 403):

```java
@SpringBootTest
@AutoConfigureMockMvc
class FundCollectionControllerTest {

    @Autowired MockMvc mockMvc;
    @Autowired JwtUtil jwtUtil;

    @Test
    @DisplayName("TC12: Member không tạo được khoản thu")
    void member_cannot_create_collection() throws Exception {
        // Setup: tạo user, lớp, đặt member vào lớp
        Long memberUserId = 3L;
        Long classroomId = 1L;
        String token = jwtUtil.generateToken(memberUserId, "member@test.com");

        mockMvc.perform(post("/api/fund/collections")
                .header("Authorization", "Bearer " + token)
                .contentType("application/json")
                .content("""
                  {"classroomId":1,"title":"Test","amount":1000}
                """))
            .andExpect(status().isForbidden())
            .andExpect(jsonPath("$.message").value(
                "Chỉ Ban cán sự được phép thao tác"));
    }
}
```

### Cấu hình DB test
Dùng **H2 in-memory** cho test (không động đến MySQL prod):
```properties
# application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
```

→ Mỗi test có DB sạch. Đảm bảo data isolated.

## 10.6. Test trên Flutter

### 10.6.1. Test thủ công theo demo script

Xem `12-demo-script.md` — 13 bước E2E từ đăng ký đến check-in sự kiện (bao gồm bước 4a cấu hình tài khoản ngân hàng trước khi tạo khoản thu).

### 10.6.2. Test đổi lớp trong workspace
Điều kiện: user đã tham gia ít nhất 2 lớp.

1. Đăng nhập user có lớp A và lớp B.
2. Vào lớp A từ `HomeScreen`.
3. Bấm tên lớp ở header của `ClassroomDetailScreen`.
4. Kiểm tra bottom sheet hiển thị danh sách lớp; lớp A có trạng thái `Đang chọn`.
5. Chọn lớp B.
6. Kiểm tra màn chuyển sang `ClassroomDetailScreen` của lớp B và tab mặc định là Tổng quan.
7. Mở tab Quỹ/Sự kiện, kiểm tra dữ liệu dùng đúng `classroomId` lớp B.
8. Bấm tên lớp và chọn lại lớp A, kiểm tra chuyển về đúng lớp A.

### 10.6.3. Test trên 2 thiết bị / emulator
Để demo "Admin xác nhận → Member thấy polling cập nhật" trực quan:
- Thiết bị 1: đăng nhập Admin, mở `CollectionPaymentsScreen`.
- Thiết bị 2: đăng nhập Member, mở `PaymentQrScreen` (đang polling).
- Trên thiết bị 1, bấm Xác nhận → trong ≤ 5s thiết bị 2 chuyển trạng thái.

### 10.6.4. Test NotificationScreen và Home badge
Điều kiện: lớp K64KTPM3 có admin và ít nhất 1 member.

1. Đăng nhập Admin, tạo sự kiện hoặc khoản thu mới.
2. Đăng nhập Member, quay về `HomeScreen`, kiểm tra icon chuông có badge unread.
3. Tap icon chuông, kiểm tra `NotificationScreen` hiển thị item mới.
4. Tap item, kiểm tra dot unread biến mất.
5. Quay lại `HomeScreen`, kiểm tra badge được refresh.
6. Tạo thêm nhiều notification, mở lại `NotificationScreen`, bấm "Đánh dấu tất cả đã đọc" và kiểm tra badge về 0.

### 10.6.5. Flutter widget test (chưa làm)
Có thể viết widget test cho:
- `LoginScreen` render đủ 2 field + nút submit.
- `HomeScreen` hiển thị "Chưa tham gia lớp nào" khi list rỗng.
- `ClassroomBankAccountScreen` validation không cho submit khi chưa chọn ngân hàng.
- Bottom sheet search ngân hàng filter đúng khi gõ.
- `NotificationScreen` render loading/empty/error/list và cập nhật dot unread sau khi mark read.

→ Đưa vào hướng phát triển.

## 10.7. Edge case nên test trước demo

| # | Edge case | Cách test |
|---|---|---|
| 1 | Token hết hạn (24h) | Sửa `jwt.expiration=60000` (1 phút) trong properties, login, đợi 1 phút, gọi API → 401 |
| 2 | DB mất kết nối khi đang dùng app | Tắt MySQL → bấm action → thấy lỗi 500 nhẹ nhàng, không crash app |
| 3 | Mất mạng khi polling QR | Bật airplane mode trong lúc đợi xác nhận → polling fail silent, status không update |
| 4 | Hai tab cùng confirm | Mở 2 Postman, gửi `PUT /confirm` đồng thời → 1 thành công, 1 nhận lỗi 400 idempotency |
| 5 | Số tiền decimal | amount=99.99 → lưu chính xác, không bị float error |
| 6 | Tiếng Việt unicode | Tạo lớp tên "Lớp Công nghệ phần mềm" → vẫn lưu + hiển thị đúng |

## 10.8. Bug đã tìm thấy + đã fix (lịch sử)

| Ngày | Bug | Fix |
|---|---|---|
| 2026-05-15 | `permitAll()` toàn bộ `/api/**` — JWT không validate | Tạo `JwtAuthenticationFilter` + `SecurityConfig` thay đổi (B1) |
| 2026-05-15 | Member gọi được API admin | `AuthorizationService.requireMember/requireAdmin` (B2) |
| 2026-05-15 | `confirmPayment` không lưu ai xác nhận | Thêm `FundPayment.confirmedBy` (B3) |
| 2026-05-15 | `checkIn` không lưu ai check-in | Thêm `EventParticipant.checkedBy` (B4) |
| 2026-05-15 | `amount` không validate > 0 | `@DecimalMin("0.01")` (B5) |
| 2026-05-15 | Member join trễ không có payment | Backfill ở `joinClassroom` (B6) |
| 2026-05-15 | Lỗi trả stacktrace HTML | Mở rộng `GlobalExceptionHandler` (B7) |
| 2026-05-15 | CORS rời rạc `@CrossOrigin` | CORS toàn cục `SecurityConfig` (B8) |

## 10.9. Tổng kết kiểm thử

- **53 test case đặc tả** bao phủ 7 module (thêm 5 test case cho Camera Check-in và 13 test case cho Notification MVP), 4 loại HTTP status (200, 400, 401, 403).
- **Manual test** đã chạy luồng E2E qua demo script (BE compile + FE pub get sạch).
- **Camera Check-in** (đặc biệt TC-FILE-01 đến TC-FILE-05): đã implement BE+FE, build/analyze pass; **cần kiểm thử thực tế trên Android device** cho luồng chụp ảnh → upload → duyệt/từ chối.
- **Notification MVP** (TC-NOTI-01 đến TC-NOTI-13): tập trung vào inbox in-app, unread count, mark read, mark all read, auto emit khi tạo event/khoản thu và ownership security.
- **Test tự động** chưa implement — đưa vào kế hoạch giai đoạn tiếp theo.
- **Bug history**: 8 bug critical đã fix toàn bộ trong audit B1–B8 + bug Camera Check-in `contentType` (FE đã thêm `MediaType` khi upload multipart).
