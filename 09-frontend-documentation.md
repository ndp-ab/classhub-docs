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

Tổng **13 màn hình** chia 3 nhóm:

### 9.4.1. Auth (2 màn)
| File | Mô tả |
|---|---|
| `screens/login_screen.dart` | Form email + password. Gọi `AuthProvider.login`. Thành công → `HomeScreen`. |
| `screens/signup_screen.dart` | Form fullName + email + password. Gọi `AuthProvider.register`. Sau register cũng vào `HomeScreen`. |

### 9.4.2. Classroom (3 màn)
| File | Mô tả |
|---|---|
| `screens/home_screen.dart` | Class selector screen sau đăng nhập. Header ClassHub + greeting, 2 quick actions Tạo lớp / Nhập mã lớp, danh sách classroom cards. Tap card lớp → mở `ClassroomDetailScreen`. |
| `screens/create_classroom_screen.dart` | Form tạo lớp, sau khi thành công hiển thị invite code + nút Copy. |
| `screens/join_classroom_screen.dart` | Input invite code, gọi `joinClassroom`. |

`HomeScreen` chỉ đóng vai trò chọn lớp để tiếp tục làm việc, không phải dashboard tổng. Dashboard/nghiệp vụ theo từng lớp nằm ở `ClassroomDetailScreen` và các tab con.

### 9.4.3. Chi tiết lớp + 3 phân hệ (8 màn)
| File | Mô tả |
|---|---|
| `screens/classroom_detail_screen.dart` | Workspace theo lớp với header compact dùng chung, bottom navigation: Tổng quan, Quỹ, Sự kiện, Thành viên. Identity card lớp + stat cards chỉ hiển thị trong tab Tổng quan. |
| `screens/fund/fund_tab.dart` | Tab "Khoản thu". Member thấy section "Khoản của bạn" + nút "Xem QR". Admin thấy FAB tạo đợt thu, tap card → chi tiết payments. |
| `screens/fund/payment_qr_screen.dart` | Hiển thị QR (Image.network), nội dung CK + nút Copy. Polling status 5s/lần qua `Timer.periodic`. Dispose timer khi pop. |
| `screens/fund/collection_payments_screen.dart` | Admin xem ai đã/chưa đóng. Nút "Xác nhận" có confirm dialog. |
| `screens/fund/create_collection_screen.dart` | Form tạo đợt thu (title, amount, deadline với DatePicker). |
| `screens/fund/expenses_screen.dart` | Tab Khoản chi: list + tổng chi. FAB cho Admin. |
| `screens/fund/create_expense_screen.dart` | Form tạo khoản chi. |
| `screens/events/events_tab.dart` | Tab Sự kiện. Member nút Đăng ký/Huỷ. Admin FAB tạo + nút "Người tham gia". |
| `screens/events/create_event_screen.dart` | Form sự kiện (DateTimePicker). |
| `screens/events/event_participants_screen.dart` | Admin xem danh sách + nút Check-in (có confirm dialog). |

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

4 service:

### `services/auth_service.dart`
- `register(fullName, email, password)` → `POST /api/auth/register`
- `login(email, password)` → `POST /api/auth/login`
- KHÔNG cần token (public endpoint).

### `services/classroom_service.dart`
- `createClassroom(className, faculty, academicYear, userId)`
- `joinClassroom(inviteCode, userId)`
- `getMyClassrooms(userId)`
- Tất cả gửi `Authorization: Bearer <token>` (đọc từ SharedPreferences).

### `services/fund_service.dart` (9 method)
- `getCollections / createCollection / getCollectionPayments / confirmPayment`
- `getMyPayments / getPaymentQr / getPaymentStatus`
- `getExpenses / createExpense`

### `services/event_service.dart` (7 method)
- `getEvents / createEvent`
- `volunteer / cancelVolunteer`
- `getParticipants / checkIn`
- `getMyEvents`

### Conventions service
- Helper `_headers(userId, {json:true})` build header (Bearer + Content-Type nếu POST).
- Helper `_errorMessage(response)` chuyển status 4xx/5xx thành string Vietnamese.
- Mọi method trả `Future<Map<String, dynamic>>` shape:
  ```dart
  {success: true, data: <model>}
  // hoặc
  {success: false, message: "..."}
  ```
- Try/catch quanh `http.xxx` để bắt lỗi network.

## 9.7. Model

| File | Class | Parse từ |
|---|---|---|
| `models/fund_collection.dart` | `FundCollection` | `CollectionResponse` |
| `models/payment.dart` | `Payment` | `PaymentResponse` (parse cả `amount`, `deadline`, `confirmedByName`) |
| `models/expense.dart` | `Expense` | `ExpenseResponse` |
| `models/event.dart` | `ClassEvent` + `EventParticipant` | `EventResponse` + `EventParticipantResponse` (có `eventId`, `checkedByName`) |

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
| `ClassroomDetailScreen` | (không gọi, chỉ truyền data qua constructor) |
| `FundTab` | `GET /api/fund/collections/{classroomId}` + `GET /api/fund/payments/my/{classroomId}` |
| `CreateCollectionScreen` | `POST /api/fund/collections` |
| `CollectionPaymentsScreen` | `GET /api/fund/collections/{collectionId}/payments` + `PUT /api/fund/payments/{paymentId}/confirm` |
| `PaymentQrScreen` | `GET /api/fund/payments/{paymentId}/qr` + polling `GET /api/fund/payments/{paymentId}/status` |
| `ExpensesScreen` | `GET /api/fund/expenses/{classroomId}` |
| `CreateExpenseScreen` | `POST /api/fund/expenses` |
| `EventsTab` | `GET /api/events/{classroomId}` + `GET /api/events/my/{classroomId}` + `POST /volunteer` / `DELETE /volunteer` |
| `CreateEventScreen` | `POST /api/events` |
| `EventParticipantsScreen` | `GET /api/events/{eventId}/participants` + `PUT /checkin/{userId}` |

## 9.12. Luồng điều hướng

```
Splash (main → AuthWrapper)
   │
   ├─ token có trong prefs → HomeScreen
   │       │
   │       ├─ Tap card lớp → ClassroomDetailScreen
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
   │       │             └─ Admin: FAB Tạo → CreateEventScreen
   │       │                       "Người tham gia" → EventParticipantsScreen
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
- Header gồm title `ClassHub`, icon thông báo, greeting `Xin chào, {fullName}` và subtitle `Chọn lớp để tiếp tục quản lý`.
- Quick actions dùng shared button: `Tạo lớp` và `Nhập mã lớp`.
- Section `Lớp học của bạn` hiển thị danh sách classroom cards.
- Mỗi card dùng `AppCard`, hiển thị className, faculty/academicYear, role badge Admin/Member, invite code và chevron để thể hiện có thể bấm.

### ClassroomDetailScreen workspace
`ClassroomDetailScreen` là workspace theo lớp sau khi chọn lớp:
- `_DashboardHeader` là header compact dùng chung cho toàn bộ tab.
- Identity card lớp và 3 stat cards chỉ nằm trong tab Tổng quan.
- Các tab Quỹ/Sự kiện/Thành viên tập trung vào nội dung chính, không lặp lại card thông tin lớp.

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

`baseUrl` được hardcode trong 4 service file. Phải đổi tay:

| Thiết bị | baseUrl |
|---|---|
| Chrome web / Desktop | `http://localhost:8080/api` |
| Android emulator | `http://10.0.2.2:8080/api` |
| Device thật (cùng WiFi) | `http://<IP-máy-tính>:8080/api` (vd `192.168.1.5`) |

→ Hướng phát triển: tách thành file `config.dart` hoặc đọc từ `.env`.

## 9.16. Limitation hiện tại

| # | Hạn chế | Giải pháp tương lai |
|---|---|---|
| 1 | Tab Thành viên là placeholder | BE thêm API `/classrooms/{id}/members` |
| 2 | Chưa có Dashboard thống kê | BE thêm API `/fund-statistics` |
| 3 | 401 chưa tự logout về Login | Thêm interceptor ở service |
| 4 | Format tiền/giờ pure-Dart (`formatVnd`, `formatDate`, `formatDateTime` ở `lib/core/utils/`), chưa dùng `intl` cho i18n | Cài `intl` + `NumberFormat`, `DateFormat` khi cần locale khác |
| 5 | Loading dùng spinner đơn (`AppLoading`) | Skeleton shimmer (package `shimmer`) |
| 6 | Chưa có sửa/xoá UI | BE thêm `PUT/DELETE` rồi FE thêm sau |
| 7 | Polling QR fail silent khi mất mạng | Thêm error message + nút retry |

## 9.17. Trạng thái build

```
$ flutter pub get
$ flutter build apk --debug
```

→ Build thành công. Test trên thiết bị thật cần đổi `baseUrl`.
