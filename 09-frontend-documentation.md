# 09 — Frontend Documentation

## 9.1. Tổng quan

Frontend ClassHub là **Flutter mobile app** (Dart 3.10+), pattern **Provider + Service Layer**, build cho Android (iOS sau này). Repo: `github.com/agio7/classhub-app`.

## 9.2. Cấu trúc thư mục

```
classhub_app/
├── lib/
│   ├── main.dart                       — Entry point + AuthWrapper
│   ├── core/                           — Design system (added 2026-05)
│   │   ├── theme/                      — Tokens: colors, spacing, radius,
│   │   │                                  text styles, AppTheme.lightTheme
│   │   ├── widgets/                    — Shared widgets: AppCard,
│   │   │                                  AppButton, AppInput,
│   │   │                                  AppPickerField, AppSectionTitle,
│   │   │                                  AppEmptyState, AppErrorState,
│   │   │                                  AppLoading, PaymentStatusBadge
│   │   ├── config/                     — AppConfig, API base URL
│   │   ├── constants/                  — UserRoles, role helpers
│   │   └── utils/                      — formatVnd, formatDate,
│   │                                      formatDateTime
│   ├── models/                         — Data class parse JSON
│   ├── providers/                      — State management (ChangeNotifier)
│   ├── services/                       — Gọi REST API
│   └── screens/                        — UI widgets (dùng lib/core)
├── pubspec.yaml                        — Dependencies
└── android/, ios/, web/, windows/, ... — Platform-specific (auto-generated)
```

## 9.3. Pattern kiến trúc

```
┌───────────────────────────────────────┐
│        Screen (View)                  │
│        screens/*.dart                 │
│  - StatefulWidget hoặc StatelessWidget │
│  - Hiển thị UI                        │
│  - Gọi Provider hoặc Service          │
└───────────┬───────────────────────────┘
            │ Provider.of / context.read
            ↓
┌───────────────────────────────────────┐
│      Provider (ViewModel)             │
│      providers/*.dart                 │
│  - ChangeNotifier                     │
│  - Giữ state (auth)                   │
│  - Notify listeners khi thay đổi      │
└───────────┬───────────────────────────┘
            │ uses
            ↓
┌───────────────────────────────────────┐
│       Service (API client)            │
│       services/*.dart                 │
│  - http.get/post/put/delete           │
│  - Gắn Authorization: Bearer header   │
│  - Parse response → Map hoặc Model    │
│  - Return {success, data/message}     │
└───────────────────────────────────────┘
            │
            ↓
       REST API (Spring Boot)
```

## 9.4. Danh sách màn hình

Tổng **15 màn hình** chia 3 nhóm:

### 9.4.1. Auth (2 màn)
| File | Mô tả |
|---|---|
| `screens/login_screen.dart` | Form email + password. Gọi `AuthProvider.login`. Thành công → `HomeScreen`. |
| `screens/signup_screen.dart` | Form fullName + email + password. Gọi `AuthProvider.register`. Sau register cũng vào `HomeScreen`. |

### 9.4.2. Classroom (5 màn)
| File | Mô tả |
|---|---|
| `screens/home_screen.dart` | Class selector screen sau đăng nhập. Header ClassHub + greeting, 2 quick actions Tạo lớp / Nhập mã lớp, danh sách classroom cards. Tap card lớp → mở `ClassroomDetailScreen`. |
| `screens/create_classroom_screen.dart` | Form tạo lớp, sau khi thành công hiển thị invite code + nút Copy. |
| `screens/join_classroom_screen.dart` | Input invite code, gọi `joinClassroom`. |
| `screens/classroom_switcher_sheet.dart` | Bottom sheet đổi lớp trong workspace. Load danh sách lớp bằng `ClassroomService.getMyClassrooms(...)`, chọn lớp khác thì trả classroom data cho màn hiện tại. |
| `screens/classroom_bank_account_screen.dart` | Form cấu hình tài khoản ngân hàng nhận tiền của lớp. Chỉ Admin. Load danh mục ngân hàng từ `GET /api/banks`, bottom sheet có search/filter local, loading/error/retry, nhập số TK/chủ TK/ghi chú. Confirmation dialog trước khi lưu. |
| `screens/notification_screen.dart` | **Mới (2026-06-09)**: Inbox thông báo in-app của user. Hỗ trợ cả danh sách global và danh sách theo `classroomId`. Tap item hiện chỉ đánh dấu đã đọc, chưa điều hướng sâu sang lớp/tab/đối tượng liên quan. Nút "Đánh dấu tất cả đã đọc". Dot badge đỏ cho thông báo chưa đọc. Pull-to-refresh. Loading/empty/error state. |

`HomeScreen` chỉ đóng vai trò chọn lớp để tiếp tục làm việc, không phải dashboard tổng. Dashboard/nghiệp vụ theo từng lớp nằm ở `ClassroomDetailScreen` và các tab con.

> **Header HomeScreen** hiển thị icon chuông với **badge số thông báo chưa đọc**. Tap icon chuông → mở `NotificationScreen`. Số unread được load khi `initState` và refresh sau khi quay lại từ `NotificationScreen`. Đây là in-app notification inbox, không phải push notification ngoài màn hình điện thoại.

### 9.4.3. Chi tiết lớp + 3 phân hệ (7 màn)

> **Ghi chú:** `FundTab`, `ExpensesScreen` và `EventsTab` là tab/widget con trong `ClassroomDetailScreen`, không tính là màn nghiệp vụ độc lập. Vì vậy nhóm này có **7 màn độc lập** và **3 tab/widget hỗ trợ**.

| File | Mô tả |
|---|---|
| `screens/classroom_detail_screen.dart` | Workspace theo lớp với header compact dùng chung, icon chuông notification theo lớp + badge unread theo `classroomId`, bottom navigation: Tổng quan, Quỹ, Sự kiện, Thành viên. Identity card lớp + stat cards chỉ hiển thị trong tab Tổng quan. |
| `screens/fund/fund_tab.dart` | **Tab widget "Khoản thu"**. Đầu tab hiển thị card tài khoản nhận tiền của lớp (Admin có nút Thiết lập/Cập nhật, Member chỉ xem). Member thấy section "Khoản của bạn" + nút "Xem QR". Admin thấy FAB tạo đợt thu, tap card → chi tiết payments. |
| `screens/fund/payment_qr_screen.dart` | Hiển thị QR (Image.network), **block thông tin tài khoản nhận tiền (ngân hàng, số TK, chủ TK)**, nội dung CK + nút Copy. Polling status 5s/lần qua `Timer.periodic`. Dispose timer khi pop. |
| `screens/fund/collection_payments_screen.dart` | Admin xem ai đã/chưa đóng. Nút "Xác nhận" có confirm dialog. |
| `screens/fund/create_collection_screen.dart` | Form tạo đợt thu (title, amount, deadline với DatePicker). |
| `screens/fund/expenses_screen.dart` | **Tab widget "Khoản chi"**: list + tổng chi. FAB cho Admin. |
| `screens/fund/create_expense_screen.dart` | Form tạo khoản chi. |
| `screens/events/events_tab.dart` | **Tab widget "Sự kiện"**. Member nút Đăng ký/Huỷ. Admin FAB tạo + nút "Người tham gia". **Camera Check-in (MVP):** Member đã đăng ký có nút "Chụp ảnh điểm danh" — chụp, preview và gửi minh chứng **inline trong tab này**, không có màn riêng. Chưa mở `EventDetailScreen` khi tap card. |
| `screens/events/create_event_screen.dart` | Form sự kiện (DateTimePicker). Chưa có input `minParticipants`. |
| `screens/events/event_participants_screen.dart` | Admin xem danh sách + nút Check-in thủ công (có confirm dialog). **Camera Check-in (MVP):** Admin xem ảnh minh chứng, duyệt hoặc từ chối kèm lý do **ngay trong màn này**, không có màn riêng. |

## 9.5. Provider

### `providers/auth_provider.dart`

`ChangeNotifier` giữ state authentication:

| Field | Loại | Mô tả |
|---|---|---|
| `_token` | `String?` | JWT token |
| `_userId` | `int?` | userId từ login response |
| `_fullName`, `_email` | `String?` | Thông tin hiển thị |
| `_isLoading` | `bool` | Đang gọi login/register |

Method:
- `checkAuth()` — đọc từ `SharedPreferences` khi app start
- `register(fullName, email, password)` — gọi `AuthService.register` + lưu state + prefs
- `login(email, password)` — tương tự
- `logout()` — clear state + `prefs.clear()`
- Getter `isLoggedIn => _token != null`

State được dùng trong `AuthWrapper` ở `main.dart` để chuyển giữa Login/Home tự động.

## 9.6. Service

5 service:

`baseUrl` không hard-code trong từng service. Cả 5 service dùng chung
`AppConfig.baseUrl` từ `lib/core/config/app_config.dart`.

### `services/auth_service.dart`
- `register(fullName, email, password)` → `POST /api/auth/register`
- `login(email, password)` → `POST /api/auth/login`
- KHÔNG cần token (public endpoint).

### `services/classroom_service.dart`
- `createClassroom(className, faculty, academicYear, userId)`
- `joinClassroom(inviteCode, userId)`
- `getMyClassrooms(userId)`
- **`getBankAccount(classroomId)`** → `GET /api/classrooms/{classroomId}/bank-account`
- **`getBanks()`** → `GET /api/banks`, parse `List<Bank>`.
- **`updateBankAccount(classroomId, bankBin, accountNo, accountName, note)`** → `PUT /api/classrooms/{classroomId}/bank-account`; body chỉ gửi `bankBin`, `accountNo`, `accountName`, optional `note`, không gửi `bankName`.
- Tất cả gửi `Authorization: Bearer <token>` (đọc từ SharedPreferences).

### `services/fund_service.dart` (9 method)
- `getCollections / createCollection / getCollectionPayments / confirmPayment`
- `getMyPayments / getPaymentQr / getPaymentStatus`
- `getExpenses / createExpense`

### `services/event_service.dart` (9 method hiện có)
- `getEvents / createEvent`
- `volunteer / cancelVolunteer`
- `getParticipants / checkIn`
- `getMyEvents`
- **`submitCheckinImage(eventId, imagePath, userId)`** → `POST /api/events/{eventId}/checkin-submissions` với `http.MultipartFile.fromPath('file', imagePath, contentType: MediaType('image', ext))` — **bắt buộc set `contentType`** để BE validate đúng.
- **`getCheckinSubmissions(eventId) / approveSubmission(submissionId) / rejectSubmission(submissionId, reason)`**

**BE contract đã sẵn sàng, FE pending:**
- Planned `getEventDetail(eventId)` → `GET /api/events/detail/{eventId}`.
- Planned `assignParticipants(eventId, userIds)` → `POST /api/events/{eventId}/participants/assign`.
- `createEvent` cần thêm optional `minParticipants` khi cập nhật `CreateEventScreen`.

### `services/notification_service.dart` (4 method)
- Dùng `AppConfig.baseUrl`, đọc token `jwt_token` từ `SharedPreferences`, gửi `Authorization: Bearer <token>`.
- **`getNotifications({page, size, classroomId})`** → `GET /api/notifications` hoặc `GET /api/notifications?classroomId={id}` — trả `List<AppNotification>`, parse Spring Page dạng `{ content: [...] }` từ response.
- **`getUnreadCount({classroomId})`** → `GET /api/notifications/unread-count` hoặc `GET /api/notifications/unread-count?classroomId={id}` — trả `int`.
- **`markAsRead(recipientId)`** → `PUT /api/notifications/{recipientId}/read`.
- **`markAllAsRead({classroomId})`** → `PUT /api/notifications/read-all` hoặc `PUT /api/notifications/read-all?classroomId={id}`.

### Conventions service
- Helper `_headers(userId, {json:true})` build header (Bearer + Content-Type nếu POST).
- Helper `_errorMessage(response)` chuyển status 4xx/5xx thành string Vietnamese.
- Các service nghiệp vụ cũ chủ yếu trả `Future<Map<String, dynamic>>` shape:
  ```dart
  {success: true, data: <model>}
  // hoặc
  {success: false, message: "..."}
  ```
- Riêng `NotificationService` trả typed value trực tiếp (`List<AppNotification>`, `int`, `void`) và throw `Exception` khi lỗi để `NotificationScreen` hiển thị loading/error state.
- Try/catch quanh `http.xxx` để bắt lỗi network.

### Role helper FE

Role được gom ở `lib/core/constants/user_roles.dart`:

```dart
class UserRoles {
  static const String admin = 'ADMIN';
  static const String member = 'MEMBER';

  static bool isAdminLike(String? role) => role == admin;
  static bool isMember(String? role) => role == member;
}
```

`HomeScreen` và `ClassroomDetailScreen` không check trực tiếp `role == 'ADMIN'`
nữa, mà dùng `UserRoles.isAdminLike(...)`. Backend hiện chỉ có `ADMIN`/`MEMBER`;
`OWNER` chưa nằm trong code FE/BE hiện tại và chỉ được ghi nhận như hướng mở rộng.

## 9.7. Model

| File | Class | Parse từ |
|---|---|---|
| `models/bank.dart` | `Bank` | `BankResponse` (`id`, `bankBin`, `code`, `name`, `shortName`, `logo`, `transferSupported`, `lookupSupported`), có getter `displayName` |
| `models/classroom_bank_account.dart` | `ClassroomBankAccount` | `ClassroomBankAccountResponse` (parse `bankBin`, `bankName`, `shortName`, `accountNo`, `accountName`, `note`, `active`, `createdByName`), có getter `displayBankName` |
| `models/fund_collection.dart` | `FundCollection` | `CollectionResponse` |
| `models/payment.dart` | `Payment` | `PaymentResponse` (parse cả `amount`, `deadline`, `confirmedByName`) |
| `models/expense.dart` | `Expense` | `ExpenseResponse` |
| `models/event.dart` | `ClassEvent` + `EventParticipant` | `EventResponse` + `EventParticipantResponse` (có `eventId`, `checkedByName`). BE đã trả thêm `minParticipants`, `source`, `assignedByName`, `assignedAt`; FE cần cập nhật model khi làm Event detail/assign. |
| `models/app_notification.dart` | `AppNotification` | `NotificationResponse` (parse `recipientId`, `notificationId`, `classroomId`, `type`, `title`, `message`, `targetType`, `targetId`, `isRead`/`read`, `readAt`, `createdAt`, `createdByName`) |

Conventions:
- Trường nullable đúng theo BE (`String?`, `int?`).
- `factory XYZ.fromJson(Map<String, dynamic> json)` parse defensive: dùng `tryParse` cho DateTime, default value cho int/bool.
- **`Payment.fromJson` parse cả `isPaid` và `paid`** — phòng Jackson serialize boolean với "is" prefix khác nhau.

## 9.8. State management — vì sao chọn Provider?

| Tiêu chí | Provider | Bloc | Riverpod |
|---|---|---|---|
| Học curve | Thấp | Cao | Trung bình |
| Boilerplate | Ít | Nhiều | Trung bình |
| Phù hợp đồ án | ✅ | Quá phức tạp | OK nhưng mới |
| Tài liệu | Phong phú | Phong phú | Mới hơn |

→ Chọn Provider vì đủ cho scope đồ án, code ngắn gọn, dễ giải thích trước hội đồng.

## 9.9. Navigation

Sử dụng **Navigator 1.0** (imperative) — không dùng GoRouter để giữ đơn giản.

| Kiểu navigation | Khi dùng |
|---|---|
| `Navigator.push` | Mở màn mới (vd Tạo lớp) |
| `Navigator.pop(context, true)` | Đóng + trả kết quả về màn cũ (để reload list) |
| `Navigator.pushAndRemoveUntil` | Logout → về Login, xoá hết stack |
| `Navigator.pushReplacement` | Đăng nhập thành công → Home (không cho back về Login) |

## 9.10. Lưu token

`SharedPreferences`:
| Key | Loại | Mô tả |
|---|---|---|
| `jwt_token` | String | JWT trả về từ login/register |
| `user_id` | int | userId |
| `full_name` | String | Hiển thị |
| `email` | String | Hiển thị |

Tất cả set ở `AuthProvider._saveAuth`, đọc ở `AuthProvider.checkAuth`, xoá ở `logout()` qua `prefs.clear()`.

## 9.11. Bảng map "Màn hình → API"

| Màn hình | API gọi |
|---|---|
| `LoginScreen` | `POST /api/auth/login` |
| `SignupScreen` | `POST /api/auth/register` |
| `HomeScreen` | `GET /api/classrooms/my` |
| `CreateClassroomScreen` | `POST /api/classrooms/create` |
| `JoinClassroomScreen` | `POST /api/classrooms/join` |
| `ClassroomDetailScreen` | Mở `ClassroomSwitcherSheet`; gọi `GET /api/notifications/unread-count?classroomId={id}` cho badge chuông theo lớp |
| `ClassroomSwitcherSheet` | `GET /api/classrooms/my` |
| `ClassroomBankAccountScreen` | `GET /api/banks` (selector ngân hàng) + `GET /api/classrooms/{classroomId}/bank-account` (load hiện tại) + `PUT /api/classrooms/{classroomId}/bank-account` (cập nhật, không gửi `bankName`) |
| `FundTab` | `GET /api/fund/collections/{classroomId}` + `GET /api/fund/payments/my/{classroomId}` + `GET /api/classrooms/{classroomId}/bank-account` (card tài khoản) |
| `CreateCollectionScreen` | `POST /api/fund/collections` |
| `CollectionPaymentsScreen` | `GET /api/fund/collections/{collectionId}/payments` + `PUT /api/fund/payments/{paymentId}/confirm` |
| `PaymentQrScreen` | `GET /api/fund/payments/{paymentId}/qr` + polling `GET /api/fund/payments/{paymentId}/status` |
| `ExpensesScreen` | `GET /api/fund/expenses/{classroomId}` |
| `CreateExpenseScreen` | `POST /api/fund/expenses` |
| `EventsTab` | `GET /api/events/{classroomId}` + `GET /api/events/my/{classroomId}` + `POST /volunteer` / `DELETE /volunteer` + **`POST /checkin-submissions`** (upload ảnh inline) |
| `CreateEventScreen` | `POST /api/events` (BE đã nhận `minParticipants`, FE chưa có input) |
| `EventParticipantsScreen` | `GET /api/events/{eventId}/participants` + `PUT /checkin/{userId}` + **`GET /checkin-submissions`** + **`PUT /checkin-submissions/{id}/approve`** + **`PUT /checkin-submissions/{id}/reject`** |
| `EventDetailScreen` | Pending FE: `GET /api/events/detail/{eventId}`, hiển thị `minParticipants` và tiến độ X/Y |
| `AssignParticipantsSheet` | Pending FE: `GET /api/classrooms/{classroomId}/members` + `POST /api/events/{eventId}/participants/assign` |
| `NotificationScreen` | Global: `GET /api/notifications`; theo lớp: `GET /api/notifications?classroomId={id}`; tap item: `PUT /api/notifications/{recipientId}/read`; mark all: `PUT /api/notifications/read-all` hoặc `?classroomId={id}` |
| `HomeScreen` (badge global) | `GET /api/notifications/unread-count` khi `initState` và sau khi quay lại từ `NotificationScreen` |
| `ClassroomDetailScreen` (badge theo lớp) | `GET /api/notifications/unread-count?classroomId={id}` khi mở lớp và sau khi quay lại từ `NotificationScreen(classroomId)` |

## 9.12. Luồng điều hướng

```
Splash (main → AuthWrapper)
   │
   ├─ token có trong prefs → HomeScreen
   │       │
   │       ├─ Tap icon chuông → NotificationScreen global (load list, tap item chỉ mark read)
   │       │
   │       ├─ Tap card lớp → ClassroomDetailScreen
   │       │       │
   │       │       ├─ Tap icon chuông → NotificationScreen(classroomId, classroomName)
   │       │       │       └─ Chỉ load notification của lớp hiện tại
   │       │       │
   │       │       ├─ Tap tên lớp ở header → ClassroomSwitcherSheet
   │       │       │       └─ Chọn lớp khác → pushReplacement ClassroomDetailScreen
   │       │       │
   │       │       ├─ Tab Tổng quan
   │       │       ├─ Tab Khoản thu (FundTab)
   │       │       │     ├─ Member: Xem QR → PaymentQrScreen
   │       │       │     └─ Admin: FAB Tạo → CreateCollectionScreen
   │       │       │              Tap card → CollectionPaymentsScreen
   │       │       ├─ Tab Khoản chi (ExpensesScreen)
   │       │       │     └─ Admin: FAB → CreateExpenseScreen
   │       │       └─ Tab Sự kiện (EventsTab)
   │       │             ├─ Member: Đăng ký / Huỷ
   │       │             ├─ Member đã đăng ký: "Chụp ảnh điểm danh" (inline, không navigate)
   │       │             └─ Admin: FAB Tạo → CreateEventScreen
   │       │                       "Người tham gia" → EventParticipantsScreen
   │       │                       (Admin xem ảnh + duyệt/từ chối ngay trong màn này)
   │       │
   │       ├─ Tạo lớp → CreateClassroomScreen
   │       ├─ Nhập mã lớp → JoinClassroomScreen
   │       └─ Logout → LoginScreen (clear stack)
   │
   └─ chưa đăng nhập → LoginScreen
            │
            └─ "Đăng ký ngay" → SignupScreen
```

## 9.13. UI Design

- **Material 3** (`useMaterial3: true`) làm foundation
- **AppTheme.lightTheme** (custom) — primary `#6366F1` (tím), success/warning/danger,
  Inter font (fallback SF Pro / system). Không dùng `colorSchemeSeed: Colors.blue`.
- Tokens (`lib/core/theme/`): `AppColors`, `AppSpacing` (8px grid),
  `AppRadius`, `AppTextStyles`.
- Tiếng Việt toàn bộ
- Icon từ `Icons` package (built-in)

### Loading / Error / Empty state
Mỗi màn có data load đều dùng shared widgets ở `lib/core/widgets/`:
- Loading → `AppLoading` (spinner + optional message, primary-color)
- Error → `AppErrorState` (icon + title + message + nút "Thử lại")
- Empty → `AppEmptyState` (icon tròn muted + title + message + CTA tuỳ chọn)

### HomeScreen class selector
Sau đăng nhập, `HomeScreen` là màn chọn lớp theo hướng workspace selector:
- Header gồm title `ClassHub`, icon thông báo in-app kèm badge unread, greeting `Xin chào, {fullName}` và subtitle `Chọn lớp để tiếp tục quản lý`.
- Quick actions dùng shared button: `Tạo lớp` và `Nhập mã lớp`.
- Section `Lớp học của bạn` hiển thị danh sách classroom cards.
- Mỗi card dùng `AppCard`, hiển thị className, faculty/academicYear, role badge Admin/Member, invite code và chevron để thể hiện có thể bấm.

### NotificationScreen
`NotificationScreen` là inbox in-app notification:
- Load danh sách global bằng `NotificationService.getNotifications(page: 0, size: 20)`.
- Khi được mở từ `ClassroomDetailScreen`, nhận `classroomId`/`classroomName` và load bằng `NotificationService.getNotifications(classroomId: ...)`, chỉ hiển thị notification của lớp hiện tại.
- Có `AppLoading`, `AppErrorState`, `AppEmptyState` và pull-to-refresh.
- Thông báo chưa đọc hiển thị dot badge đỏ; tap item gọi `markAsRead(recipientId)` và cập nhật state local.
- Nút "Đánh dấu tất cả đã đọc" gọi `markAllAsRead()` hoặc `markAllAsRead(classroomId: ...)` và set toàn bộ item trong list hiện tại sang đã đọc.
- Hiện chưa deep-link theo `targetType/targetId` sang tab/lớp/khoản thu/sự kiện; tap item chủ yếu mark read trong inbox.

### ClassroomDetailScreen workspace
`ClassroomDetailScreen` là workspace theo lớp sau khi chọn lớp:
- `_DashboardHeader` là header compact dùng chung cho toàn bộ tab.
- Header có icon chuông theo lớp, load unread bằng `NotificationService.getUnreadCount(classroomId: widget.classroomId)`.
- Tap icon chuông mở `NotificationScreen(classroomId: widget.classroomId, classroomName: widget.classroomName)`; sau khi quay lại thì refresh badge theo lớp.
- Tên lớp trong `_DashboardHeader` có thể bấm để mở `ClassroomSwitcherSheet`.
- Khi chọn lớp khác, màn dùng `Navigator.pushReplacement` mở lại `ClassroomDetailScreen` với `classroomId`, `classroomName`, `inviteCode`, `role`, `faculty`, `academicYear` của lớp mới; tab mặc định quay về Tổng quan.
- Identity card lớp và 3 stat cards chỉ nằm trong tab Tổng quan.
- Các tab Quỹ/Sự kiện/Thành viên tập trung vào nội dung chính, không lặp lại card thông tin lớp.

### ClassroomSwitcherSheet
`ClassroomSwitcherSheet` là bottom sheet đổi lớp:
- Nhận `currentClassroomId`.
- Load danh sách lớp user đã tham gia bằng `ClassroomService.getMyClassrooms(userId)`.
- Dùng `AppLoading`, `AppErrorState`, `AppEmptyState` cho loading/error/empty.
- Lớp hiện tại hiển thị trạng thái `Đang chọn` và dấu check.
- Tap lớp hiện tại chỉ đóng sheet; tap lớp khác trả classroom data qua `Navigator.pop(context, classroom)`.

### Confirm dialog
2 action không hoàn tác có confirm:
- "Xác nhận thanh toán" trong `CollectionPaymentsScreen`
- "Check-in" trong `EventParticipantsScreen`

### Shared widgets bổ sung
- `AppCard` — surface có border, tuỳ chọn `onTap`
- `AppButton` — 4 variant (primary/secondary/ghost/danger) × 3 size, có `loading` prop
- `AppInput` — text field bọc `TextFormField`, label + prefix/suffix icon
- `AppPickerField` — read-only field cho tap-to-pick (date/time/modal)
- `AppSectionTitle` — tiêu đề section
- `PaymentStatusBadge` + `paymentStatusColor/Icon/Label` helpers — semantic
  mapping UNPAID → warning, PENDING_VERIFICATION → primary, CONFIRMED → success

### Bank account setup UI
`ClassroomBankAccountScreen` dùng bottom sheet search để chọn ngân hàng:
- Danh sách ngân hàng lấy từ API `GET /api/banks`, parse vào `Bank`.
- Bottom sheet có search bar + scrollable list, filter local theo `displayName`, `name`, `code`, `bankBin`.
- Có loading/error/retry khi không tải được danh mục ngân hàng; empty state khi catalog chưa có dữ liệu.
- User **bắt buộc chọn** từ danh sách, không được nhập tự do
- Khi chọn ngân hàng → FE giữ `bankBin`; tên hiển thị lấy từ `Bank.displayName`
- Bank BIN **không hiển thị** trên UI, chỉ gửi khi submit
- Form validation: phải chọn ngân hàng trước khi submit
- Confirmation dialog hiển thị: ngân hàng đã chọn, số tài khoản, chủ tài khoản, ghi chú (không hiển thị BIN)

## 9.14. Dependencies (`pubspec.yaml`)

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.1                   # gọi REST API
  shared_preferences: ^2.2.2     # lưu token local
  provider: ^6.1.1               # state management

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
```

**Tối thiểu nhất** — không thêm dependency nào ngoài 3 cái cốt lõi.

## 9.15. baseUrl theo môi trường

`baseUrl` được quản lý tập trung tại:

```txt
lib/core/config/app_config.dart
```

Nội dung cấu hình:

```dart
class AppConfig {
  static const String baseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:8080/api',
  );
}
```

Mặc định là `http://localhost:8080/api`. Không sửa tay trong
`auth_service.dart`, `classroom_service.dart`, `fund_service.dart`,
`event_service.dart`.

| Thiết bị | Cách chạy |
|---|---|
| Chrome web / Desktop | `flutter run` |
| Android emulator | `flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8080/api` |
| Device thật (cùng WiFi) | `flutter run --dart-define=API_BASE_URL=http://<IP-LAN>:8080/api` |

Build APK demo:

```bash
flutter build apk --release --dart-define=API_BASE_URL=http://<IP-LAN>:8080/api
```

## 9.16. Limitation hiện tại

| # | Hạn chế | Giải pháp tương lai |
|---|---|---|
| 1 | Tab Thành viên là placeholder | FE tích hợp API `/classrooms/{id}/members` đã có |
| 2 | Chưa có Dashboard thống kê | BE thêm API `/fund-statistics` |
| 3 | 401 chưa tự logout về Login | Thêm interceptor ở service |
| 4 | Format tiền/giờ pure-Dart (`formatVnd`, `formatDate`, `formatDateTime` ở `lib/core/utils/`), chưa dùng `intl` cho i18n | Cài `intl` + `NumberFormat`, `DateFormat` khi cần locale khác |
| 5 | Loading dùng spinner đơn (`AppLoading`) | Skeleton shimmer (package `shimmer`) |
| 6 | Chưa có sửa/xoá UI | BE thêm `PUT/DELETE` rồi FE thêm sau |
| 7 | Polling QR fail silent khi mất mạng | Thêm error message + nút retry |
| 8 | **Chưa có UI lịch sử tài khoản ngân hàng** | FE thêm màn history dùng API `/api/classrooms/{id}/bank-account/history` |
| 9 | **Camera Check-in đã implement; cần kiểm thử thực tế trên Android device thật** | Chạy bằng `flutter run --dart-define=API_BASE_URL=http://<IP-LAN>:8080/api` và test luồng chụp ảnh → upload → duyệt |
| 10 | **`/uploads/**` hiện public — ảnh minh chứng ai cũng xem được qua URL** | Hướng phát triển: API download có JWT + phân quyền theo lớp |
| 11 | **Notification hiện là in-app inbox, chưa có Firebase FCM/push notification ngoài app** | Thêm device token + FCM khi cần push nền/ngoài màn hình điện thoại |
| 12 | **Chưa có reminder/scheduler trước giờ sự kiện** | Thêm job scheduler và notification type riêng |
| 13 | **Đã có notification theo lớp trong `ClassroomDetailScreen`; chưa có deep-link chi tiết vào đúng tab/bản ghi** | FE map `targetType/targetId` sang màn/tab tương ứng khi cần deep-link |
| 14 | **Tap notification hiện chủ yếu mark read, chưa mở chi tiết theo `targetType/targetId`** | Bổ sung điều hướng chi tiết khi có thiết kế deep-link |
| 15 | **Chưa có user notification settings, device token hoặc delivery logs** | Thêm khi triển khai push notification/notification preferences |
| 16 | **BE đã hỗ trợ Event detail + assign participants, FE chưa tích hợp** | Thêm `EventDetailScreen`, `AssignParticipantsSheet`, input `minParticipants` trong `CreateEventScreen`, tap card ở `EventsTab` mở detail |

## 9.17. Trạng thái build

```
$ flutter pub get
$ flutter build apk --debug
```

→ Build thành công. Test trên thiết bị thật cần truyền `API_BASE_URL` bằng `--dart-define`.
