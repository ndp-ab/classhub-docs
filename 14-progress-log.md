# 14 — Progress Log

> Nhật ký quá trình phát triển ClassHub.
> Dùng để trả lời câu hỏi "em làm trong bao lâu, gặp khó khăn gì" khi bảo vệ.

## Giai đoạn 1 — Khảo sát + thiết kế (Tuần 1-2)

| Việc | Trạng thái |
|---|---|
| Phỏng vấn 5 ban cán sự lớp K63–K65 | ✅ |
| Tổng hợp vấn đề thực tế (Zalo + Excel + Forms rời rạc) | ✅ |
| Định nghĩa 3 phân hệ chính (Auth/Quỹ/Sự kiện) | ✅ |
| Đăng ký đề tài + đề cương | ✅ |
| Thiết kế ERD (13 bảng — cập nhật theo quá trình phát triển) | ✅ |
| Xác định 3 actor (Guest/Member/Admin) | ✅ |
| Lập danh sách 21 endpoint dự kiến | ✅ |
| Use Case Diagram | ✅ (text-based + Mermaid) |
| Wireframe Figma | 🟡 Backfill (làm sau code) |

## Giai đoạn 2 — Backend core (Tuần 3-5)

| Việc | Trạng thái |
|---|---|
| Setup Spring Boot project + MySQL | ✅ |
| Entity + repository chính (tăng dần theo các giai đoạn, hiện có notification in-app nâng cao) | ✅ |
| Auth: register/login + BCrypt + JWT generation | ✅ |
| Classroom: create/join/list | ✅ |
| Fund collections + payments + auto-sinh | ✅ |
| QR VietQR + polling status | ✅ |
| Fund expenses | ✅ |
| Event + participant + check-in + assigned participants | ✅ |
| `GlobalExceptionHandler` cơ bản | ✅ |

**Vấn đề gặp:**
- Spring Boot 4 mới release, một số tutorial cũ không apply được.
- Đầu tiên dùng `Long` cho FK (vd `Classroom.createdBy`), sau đó refactor sang `@ManyToOne User` để chuẩn JPA.
- Lombok + IntelliJ cần plugin riêng.

## Giai đoạn 3 — Frontend core (Tuần 5-6)

| Việc | Trạng thái |
|---|---|
| Setup Flutter project + Provider | ✅ |
| 5 screen auth + classroom | ✅ |
| Service layer + SharedPreferences token | ✅ |
| Classroom detail với 4 tab | ✅ |
| Fund tab + payment QR + polling 5s | ✅ |
| Admin confirm + collection_payments_screen | ✅ |
| Expenses screen + create | ✅ |
| Event tab + create + participants | ✅ core cũ; pending detail/assign mới |
| Models có defensive parsing | ✅ |

**Vấn đề gặp:**
- Android emulator dùng `10.0.2.2` thay vì `localhost`; FE hiện dùng `AppConfig.baseUrl` và override bằng `--dart-define=API_BASE_URL=...`.
- Lombok+Jackson serialize boolean field tên `isPaid` thành `paid` — phải parse defensive cả 2 key.
- Polling Timer phải nhớ `dispose()` khi pop màn để tránh memory leak.

## Giai đoạn 4 — Audit + vá bảo mật (1 ngày — 2026-05-15)

Sau khi review theo checklist, phát hiện 8 lỗi nghiệp vụ/bảo mật quan trọng. Triển khai vá toàn bộ trong 1 phiên làm việc.

| Block | Việc | Trạng thái |
|---|---|---|
| B1 | `JwtAuthenticationFilter` + SecurityContext + JwtAuthenticationEntryPoint | ✅ |
| B2 | `AuthorizationService.requireMember/requireAdmin` gọi mọi service | ✅ |
| B3 | `FundPayment.confirmedBy` + idempotency | ✅ |
| B4 | `EventParticipant.checkedBy` | ✅ |
| B5 | `@DecimalMin("0.01")` cho amount | ✅ |
| B6 | Backfill payment khi member join muộn | ✅ |
| B7 | Mở rộng `GlobalExceptionHandler` (ForbiddenException, validation, generic) | ✅ |
| B8 | CORS toàn cục, bỏ `@CrossOrigin` rải rác | ✅ |

**Tác động lên FE:** Phải đổi 3 service Flutter để gửi `Authorization: Bearer` thay vì `X-User-Id`. Models update để parse field mới.

**Kết quả compile:** `mvnw clean compile` → 52 file, BUILD SUCCESS, 1 warning vô hại (deprecated `WebAuthenticationDetailsSource`).

**Chi tiết:** `classhub-api/documents/BACKEND_FIX_LOG.md` + `BACKEND_AUDIT.md`.

## Giai đoạn 5 — Polish UX FE + Documentation (Tuần 7)

| Việc | Trạng thái |
|---|---|
| Confirm dialog trước "Xác nhận thanh toán" | ✅ |
| Confirm dialog trước "Check-in" | ✅ |
| Hiển thị amount + deadline trong "Khoản của bạn" | ✅ |
| Match my-events qua eventId (bỏ workaround title) | ✅ |
| Pull-to-refresh các list | ✅ (đã có sẵn) |
| Error retry button (fund_tab, events_tab) | ✅ |
| Documentation (14 file) | ✅ |

## Giai đoạn 6 — Tài liệu (đang làm)

| Việc | Trạng thái |
|---|---|
| 14 file Markdown trong `docs/` | ✅ |
| Báo cáo đồ án (Word) | ⬜ |
| Slide bảo vệ | ⬜ |
| Use Case Diagram chuẩn UML (vẽ tay/Visio) | ⬜ |
| Sequence Diagram chuẩn UML | 🟡 Đã có dạng Mermaid |
| ERD chuẩn UML | 🟡 Đã có dạng Mermaid |
| Postman collection export | ⬜ |

## Giai đoạn 8 — Design system + UI polish (2026-05-22 → 2026-05-23)

Refactor toàn bộ FE sang design system tách rời để chuẩn hoá UI, đồng thời
khắc phục 1 bug treo UI nghiêm trọng trên Chrome Web.

| Việc | Trạng thái |
|---|---|
| `lib/core/theme/` — tokens (colors, spacing, radius, text styles) + AppTheme.lightTheme | ✅ |
| `lib/core/widgets/` — AppCard, AppButton, AppInput, AppPickerField, AppSectionTitle, AppEmptyState, AppErrorState, AppLoading, PaymentStatusBadge | ✅ |
| `lib/core/utils/formatters.dart` — formatVnd, formatDate, formatDateTime | ✅ |
| Tier 1–3: refactor 6 màn (classroom_detail, create_expense/collection/event, expenses, event_participants) | ✅ |
| Tier 4: refactor 4 màn lớn (collection_payments, events_tab, fund_tab, payment_qr) | ✅ |
| Fix overflow EventsTab card (Flexible + ellipsis) | ✅ |
| Fix bug freeze CollectionPaymentsScreen trên Chrome Web (tap đợt thu → UI đứng + RAF violations 100–500ms/frame) — root cause: widget tree gốc Card+ListTile+ElevatedButton trailing + RefreshIndicator+ListView với spread operator nested. Refactor sang shared widgets giải freeze. | ✅ |
| Merge về main + push origin/main | ✅ |

**Vấn đề gặp:**
- Bug freeze CPS không tái hiện qua đọc code; phải làm debug branch stub body để confirm khu vực gây freeze nằm trong widget tree, không phải lifecycle. Sau đó refactor Tier 4 sớm hơn dự kiến để fix.
- Pre-existing lint (`_subtitle`, `unnecessary_underscores`, `withOpacity` deprecated) được clean kèm refactor — không phải pass riêng.

## Giai đoạn 8.1 — Refactor HomeScreen UI (2026-05-31)

Refactor riêng `HomeScreen` thành class selector screen sau đăng nhập. Màn này tập trung vào chọn lớp, không đóng vai trò dashboard tổng; dashboard theo lớp vẫn nằm ở `ClassroomDetailScreen`.

| Việc | Trạng thái |
|---|---|
| Header ClassHub + greeting `Xin chào, {fullName}` + subtitle chọn lớp | ✅ |
| Quick actions `Tạo lớp` / `Nhập mã lớp` dùng design system | ✅ |
| Classroom card dùng `AppCard`, hiển thị tên lớp, khoa/năm học, role badge, mã lớp, chevron | ✅ |
| Empty/loading/error dùng `AppEmptyState`, `AppLoading`, `AppErrorState` | ✅ |

**Phạm vi:** Chỉ refactor UI trong `lib/screens/home_screen.dart`; không đổi business logic, service/API, model/provider hoặc navigation flow.

**Kiểm tra:** `flutter analyze lib/screens/home_screen.dart` → passed. `flutter analyze` toàn project → passed.

## Giai đoạn 8.2 — Điều chỉnh ClassroomDetailScreen workspace (2026-06-01)

Điều chỉnh layout `ClassroomDetailScreen`: `_DashboardHeader` vẫn dùng chung toàn bộ tab, còn identity card lớp và 3 stat cards chỉ hiển thị trong tab Tổng quan. Các tab Quỹ/Sự kiện/Thành viên tập trung vào nội dung chính, không lặp lại card thông tin lớp.

**Phạm vi:** Chỉ refactor layout UI trong `lib/screens/classroom_detail_screen.dart`; không đổi service/model/provider/API hoặc logic của `FundTab`, `ExpensesScreen`, `EventsTab`.

**Kiểm tra:** `flutter analyze` → passed.

## Giai đoạn 8.3 — Centralize role checks FE (2026-06-03)

Gom logic role ở frontend vào `lib/core/constants/user_roles.dart` để tránh mỗi
màn tự check string khác nhau.

| Việc | Trạng thái |
|---|---|
| Thêm `UserRoles.admin`, `UserRoles.member` | ✅ |
| Thêm `UserRoles.isAdminLike(role)` cho quyền admin | ✅ |
| `HomeScreen` dùng `UserRoles.isAdminLike(...)` thay vì `role == 'ADMIN'` | ✅ |
| `ClassroomDetailScreen` dùng `UserRoles.isAdminLike(...)` thay vì check role string trực tiếp | ✅ |

**Phạm vi:** Chỉ đổi logic role ở FE; không đổi service/API, payment status,
polling hoặc UI layout.

**Ghi chú:** `OWNER` chưa có trong BE/FE hiện tại, chỉ là hướng mở rộng sau này.

**Kiểm tra:** `flutter analyze` → passed.

## Giai đoạn 8.4 — Classroom switcher sheet FE (2026-06-03)

Bổ sung luồng đổi lớp ngay trong `ClassroomDetailScreen` để user không phải quay
về Home khi muốn chuyển workspace.

| Việc | Trạng thái |
|---|---|
| Thêm `lib/screens/classroom_switcher_sheet.dart` | ✅ |
| Bottom sheet load danh sách lớp bằng `ClassroomService.getMyClassrooms(...)` | ✅ |
| Lớp hiện tại hiển thị `Đang chọn` + dấu check | ✅ |
| Chọn lớp khác dùng `Navigator.pushReplacement` mở lại `ClassroomDetailScreen` | ✅ |

**Phạm vi:** Chỉ đổi frontend; không thêm backend API, không sửa service API.

**Kiểm tra:** `flutter analyze` → passed.

## Giai đoạn 8.5 — Tài khoản ngân hàng theo lớp (2026-06-05)

Thêm tính năng tài khoản ngân hàng cấu hình theo từng lớp, QR động lấy từ DB thay vì dùng tài khoản cố định.

### Backend
| Việc | Trạng thái |
|---|---|
| Thêm entity `ClassroomBankAccount` (9 entity) | ✅ |
| Thêm repository `ClassroomBankAccountRepository` | ✅ |
| Thêm service `ClassroomBankAccountService` (getBankAccount, upsert, getHistory) | ✅ |
| Thêm controller `ClassroomBankAccountController` | ✅ |
| Thêm DTO `UpdateClassroomBankAccountRequest`, `ClassroomBankAccountResponse` | ✅ |
| Cập nhật `FundCollectionService.createCollection()` kiểm tra bank account | ✅ |
| Cập nhật `FundCollectionService.generateQr()` lấy bank account từ DB | ✅ |
| Tắt SQL logging (`spring.jpa.show-sql=false`) | ✅ |
| Comment config tài khoản cố định trong `application.properties` | ✅ |

### Frontend
| Việc | Trạng thái |
|---|---|
| Thêm model `ClassroomBankAccount` | ✅ |
| Thêm constants `lib/core/constants/vietnam_banks.dart` (20 ngân hàng) | ✅ lịch sử ban đầu, đã thay bằng Bank catalog dynamic ở giai đoạn 8.9 |
| Thêm màn `ClassroomBankAccountScreen` với bottom sheet search ngân hàng | ✅ |
| Cập nhật `ClassroomService`: thêm `getBankAccount`, `updateBankAccount` | ✅ |
| Cập nhật `FundTab`: card tài khoản nhận tiền ở đầu tab | ✅ |
| Cập nhật `PaymentQrScreen`: hiển thị block tài khoản nhận tiền | ✅ |
| Confirmation dialog trước khi lưu tài khoản | ✅ |
| Ẩn Bank BIN khỏi UI, user chỉ chọn ngân hàng từ list | ✅ |

### Validation & Testing
| Việc | Trạng thái |
|---|---|
| Backend compile: `./mvnw clean compile` → BUILD SUCCESS | ✅ |
| Frontend analyze: `flutter analyze` → No issues found | ✅ |
| Cập nhật tài liệu (8 file .md) | ✅ |

**Phạm vi:** BE thêm 3 API mới (GET/PUT/GET history), FE thêm 1 màn + 1 constants file + cập nhật 2 màn hiện có.

**Hướng phát triển tại thời điểm 2026-06-05:** API `/api/banks` dynamic, logo ngân hàng, UI lịch sử tài khoản, unique constraint DB cho (classroom_id, active=true). Phần Bank catalog dynamic đã hoàn thành ở giai đoạn 8.9.

## Giai đoạn 8.6 — Camera Check-in (2026-06-07)

Thêm chức năng chụp ảnh minh chứng điểm danh sự kiện.

### Backend
| Việc | Trạng thái |
|---|---|
| Thêm entity `EventCheckinSubmission` (10 entity) | ✅ |
| Thêm repository `EventCheckinSubmissionRepository` | ✅ |
| Thêm service `FileStorageService` (validate contentType/extension/size, lưu file) | ✅ |
| Thêm service `EventCheckinSubmissionService` (submit, list, approve, reject) | ✅ |
| Thêm controller `EventCheckinSubmissionController` (4 endpoint) | ✅ |
| Thêm DTO `EventCheckinSubmissionResponse`, `RejectCheckinRequest` | ✅ |
| Cấu hình `classhub.upload-dir` trong `application.properties` | ✅ |
| `WebMvcConfig.addResourceHandlers` expose `/uploads/**` | ✅ |

### Frontend
| Việc | Trạng thái |
|---|---|
| Thêm dependency `image_picker`, `http_parser`, `path` vào `pubspec.yaml` | ✅ |
| Cập nhật `event_service.dart`: thêm `submitCheckinImage` với `contentType: MediaType('image', ext)` | ✅ |
| Cập nhật `events_tab.dart`: nút "Chụp ảnh điểm danh" + luồng chụp/preview/gửi **inline** (không tạo màn riêng) | ✅ |
| Cập nhật `event_participants_screen.dart`: hiển thị trạng thái + xem ảnh + duyệt/từ chối **inline** | ✅ |

### Bug debug
| Bug | Nguyên nhân | Fix |
|---|---|---|
| FE: "File phải là ảnh" dù đã chụp ảnh | `MultipartFile.fromPath` không set `contentType` → BE nhận `null`/`octet-stream` | Thêm `contentType: MediaType('image', ext)` — `http_parser` package |

### Validation & Testing
| Việc | Trạng thái |
|---|---|
| Backend compile: `./mvnw clean compile` → BUILD SUCCESS (58 file) | ✅ |
| Frontend analyze: `flutter analyze` → No issues found | ✅ |
| Deploy APK trên Xiaomi HyperOS | ✅ |
| Test E2E trên Android device thật | ⚠️ Cần kiểm thử |

## Giai đoạn 8.7 — In-app Notification nâng cao (2026-06-09)

Hoàn thành thông báo **in-app** nâng cao: backend sinh notification theo nghiệp vụ, hỗ trợ filter theo lớp, frontend có inbox thông báo global/theo lớp, badge unread ở `HomeScreen` và badge unread theo lớp trong `ClassroomDetailScreen`.

### Backend
| Việc | Trạng thái |
|---|---|
| Thêm entity `Notification`, `NotificationRecipient` | ✅ |
| Thêm enum `NotificationType`, `NotificationTargetType` | ✅ |
| Thêm repository `NotificationRepository`, `NotificationRecipientRepository` | ✅ |
| Thêm DTO `NotificationResponse`, `UnreadCountResponse` | ✅ |
| Thêm `NotificationService`: list, unread-count, mark-read, mark-all-read, createNotification; list/count/read-all hỗ trợ optional `classroomId` | ✅ |
| Thêm `NotificationController` prefix `/api/notifications` với 4 route mapping và các biến thể filter `classroomId` | ✅ |
| Dùng `SecurityUtil.currentUserId()`, không nhận `userId` từ client | ✅ |
| `markAsRead` dùng `recipientId + currentUserId` để chặn đọc/sửa recipient của user khác | ✅ |
| `createNotification(...)` dùng `REQUIRES_NEW` transaction | ✅ |

### Frontend
| Việc | Trạng thái |
|---|---|
| Thêm model `AppNotification` | ✅ |
| Thêm `NotificationService` gọi API notification bằng JWT token, hỗ trợ optional `classroomId` | ✅ |
| Thêm `NotificationScreen` với loading/error/empty/list, pull-to-refresh, tap item mark read; hỗ trợ global inbox và inbox theo lớp | ✅ |
| Thêm nút "Đánh dấu tất cả đã đọc" | ✅ |
| Cập nhật `HomeScreen`: icon chuông + badge unread, refresh count sau khi quay lại từ `NotificationScreen` | ✅ |
| Cập nhật `ClassroomDetailScreen`: icon chuông theo lớp + badge unread theo `classroomId`, mở `NotificationScreen` theo lớp | ✅ |

### Auto emit
| Luồng | Trạng thái |
|---|---|
| Admin tạo sự kiện → member trong lớp nhận `EVENT_CREATED`, trừ người tạo | ✅ |
| Admin tạo khoản thu → member trong lớp nhận `COLLECTION_CREATED`, trừ người tạo | ✅ |
| Admin xác nhận payment → member sở hữu payment nhận `PAYMENT_CONFIRMED` | ✅ |
| Member submit ảnh check-in → admin/lớp trưởng nhận `CHECKIN_SUBMITTED` | ✅ |
| Admin duyệt ảnh check-in → member chủ submission nhận `CHECKIN_APPROVED` | ✅ |
| Admin từ chối ảnh check-in → member chủ submission nhận `CHECKIN_REJECTED` | ✅ |

**Phạm vi chưa làm:** Chưa có Firebase FCM, push notification ngoài app, reminder/scheduler trước giờ sự kiện, user notification settings, device token hoặc delivery logs. Tap notification hiện chủ yếu mark read và xem inbox, chưa deep-link chi tiết sang đúng lớp/tab/khoản thu/sự kiện/bản ghi.

## Giai đoạn 8.8 — Nâng cấp nghiệp vụ Event: minParticipants + assigned participants (2026-06-12)

Backend đã hoàn thành nâng cấp nghiệp vụ Event để Admin theo dõi số lượng tối thiểu và chỉ định thêm thành viên khi chưa đủ người.

### Backend
| Việc | Trạng thái |
|---|---|
| Thêm `Event.minParticipants`, `CreateEventRequest.minParticipants`, `EventResponse.minParticipants` | ✅ |
| Thêm `EventParticipant.source`, `assignedBy`, `assignedAt` và enum `ParticipantSource` | ✅ |
| Volunteer tạo participant `source=VOLUNTEER`; data cũ `source == null` fallback thành `VOLUNTEER` | ✅ |
| Thêm `GET /api/events/detail/{eventId}` cho member của lớp chứa event | ✅ |
| Thêm `POST /api/events/{eventId}/participants/assign` cho Admin lớp | ✅ |
| Assign idempotent: distinct duplicate `userIds`, skip user đã tham gia, không đổi `VOLUNTEER` sang `ASSIGNED` | ✅ |
| Cancel volunteer chặn participant `ASSIGNED`; `VOLUNTEER` chưa check-in vẫn huỷ được | ✅ |
| Verification: `mvnw.cmd clean compile` BUILD SUCCESS | ✅ |
| Verification: `mvnw.cmd test` pass, 1 test, 0 failures/errors | ✅ |

### Frontend pending
| Việc | Trạng thái |
|---|---|
| `EventDetailScreen` hiển thị detail, `minParticipants`, tiến độ X/Y và còn thiếu N | 🟡 Pending |
| `AssignParticipantsSheet` lấy danh sách member và gọi API assign | 🟡 Pending |
| `CreateEventScreen` thêm input `minParticipants` | 🟡 Pending |
| `EventsTab` tap card mở detail thay vì chỉ thao tác inline | 🟡 Pending |

## Giai đoạn 8.9 — Bank catalog VietQR dynamic (2026-06-12)

Thay danh sách ngân hàng hard-code ở FE bằng danh mục ngân hàng đồng bộ từ VietQR và lưu trong backend.

### Backend
| Việc | Trạng thái |
|---|---|
| Thêm entity `Bank`, bảng `banks` với `bin` unique, `code`, `name`, `shortName`, `logo`, support flags, `active` | ✅ |
| Thêm `BankRepository`, `BankService`, `BankController` | ✅ |
| Thêm DTO `BankResponse`, `VietQrBanksResponse` | ✅ |
| Thêm `GET /api/banks` trả bank active, response dùng `bankBin` | ✅ |
| Thêm `POST /api/banks/sync` gọi `vietqr.banks-url=https://api.vietqr.io/v2/banks` và upsert theo `bin` | ✅ |
| `ClassroomBankAccountService` validate `bankBin` qua `BankRepository`, snapshot `bankName`/`shortName` vào `classroom_bank_accounts` | ✅ |
| `UpdateClassroomBankAccountRequest` chỉ còn `bankBin`, `accountNo`, `accountName`, `note`; không nhận `bankName` | ✅ |

### Frontend
| Việc | Trạng thái |
|---|---|
| Thêm model `Bank` | ✅ |
| Thêm `ClassroomService.getBanks()` gọi `GET /api/banks` | ✅ |
| `ClassroomBankAccountScreen` load bank từ API, có loading/error/retry, search/filter local | ✅ |
| Xóa `lib/core/constants/vietnam_banks.dart` | ✅ |
| `updateBankAccount()` không gửi `bankName` | ✅ |
| `FundTab` hiển thị tài khoản bằng `account.displayBankName` | ✅ |

### Limitation
| Việc | Trạng thái |
|---|---|
| `POST /api/banks/sync` hiện chỉ protected bằng JWT vì chưa có role system-admin/global-admin | ⚠️ MVP limitation |
| Production nên restrict endpoint sync hoặc chuyển thành job nội bộ/scheduler | 🟢 Hướng phát triển |

## Giai đoạn 7 — Bổ sung trước demo (kế hoạch)

| Việc | Ưu tiên | Effort |
|---|---|---|
| API `/classrooms/{id}/members` | ✅ Đã có BE | FE có thể dùng cho tab Thành viên/assign participant |
| API `/fund-statistics` | 🔴 Cao | 1 tiếng |
| FE tab Thành viên (sau khi BE có API) | 🟡 | 30 phút |
| FE Dashboard quỹ (widget tổng quan) | 🟡 | 1-2 tiếng |
| Smoke test BE bằng Postman | 🔴 | 15 phút |
| Test E2E 11 bước demo trên 2 thiết bị | 🟡 | 30 phút |
| Đổi `vietqr.account-no` thành TK thật | ~~đã obsolete: config cố định đã bị comment ra từ giai đoạn 8.5~~ | — |
| `@Future` validate deadline/eventTime | 🟡 | 5 phút |
| API edit/delete collection/event/expense | 🟢 | 1 tiếng |
| Test case JUnit + MockMvc | 🟢 | 3-4 tiếng |

## Tóm tắt timeline

```
Tuần 1-2:        Khảo sát + thiết kế
Tuần 3-5:        Backend core
Tuần 5-6:        Frontend core
2026-05-15:      Audit + vá B1-B8 (1 ngày)
Tuần 7:          Polish UX + Documentation
2026-05-22→23:   Design system rollout + fix freeze CPS
2026-05-31:      Refactor HomeScreen thành class selector
2026-06-01:      Điều chỉnh ClassroomDetailScreen workspace
2026-06-03:      Classroom switcher sheet trong workspace
2026-06-05:      Tài khoản ngân hàng theo lớp
2026-06-07:      Camera Check-in (ảnh minh chứng điểm danh)
2026-06-09:      In-app Notification nâng cao (global/class inbox + unread badge)
2026-06-12:      Event minParticipants + assigned participants (BE xong, FE pending)
2026-06-12:      Bank catalog VietQR dynamic (BE+FE xong)
Trước demo:      Statistics + smoke test + test E2E Camera Check-in trên device + FE Event detail/assign nếu kịp
```

## Số liệu thống kê

| Hạng mục | Số |
|---|---|
| File source BE (Java) | ~78 |
| File source FE (Dart) | ~50 |
| Endpoint REST | 38 |
| Bảng DB | 13 |
| Screen Flutter | 15 (Camera Check-in là inline UI; Notification có `NotificationScreen` riêng) |
| File documentation | 14 |
| Test case đặc tả | 74 |
| Bug critical đã fix | 10 (B1–B8 + freeze CollectionPaymentsScreen + Camera Check-in contentType) |
| Tổng thời gian phát triển | ~7 tuần + 2 ngày design system polish + 1 ngày bank account feature + 1 ngày camera check-in + 1 ngày notification in-app nâng cao + 1 ngày nâng cấp Event BE + Bank catalog dynamic |

## Bài học rút ra

1. **Audit sớm hơn**: Nếu audit theo checklist từ tuần 4 thay vì tuần 6 thì không phải refactor nhiều ở B1+B2.
2. **Test sớm hơn**: Manual test với Postman từng phân hệ ngay sau khi viết — không phải để tới khi tích hợp FE mới phát hiện bug.
3. **Lưu thông tin về quyết định thiết kế**: Tại sao chọn Provider thay Bloc, tại sao backend hiện chưa có OWNER role... Để trả lời hội đồng dễ.
4. **Hibernate `ddl-auto=update`** rất tiện nhưng phải backup DB trước khi đổi entity quan trọng.
5. **Confirm dialog** cho action destructive đáng làm — chỉ 10 dòng code nhưng chống lỗi UX rõ rệt.
