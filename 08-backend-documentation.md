# 08 — Backend Documentation

## 8.1. Tổng quan

Backend ClassHub là **Spring Boot 4.0** project, viết bằng **Java 17**, build bằng **Maven**, dùng **MySQL 8** làm database. Repo: `github.com/agio7/classhub-api`.

## 8.2. Cấu trúc package

```
com.classhub.classhubapi/
├── ClasshubApiApplication.java       — main class với @SpringBootApplication
├── config/                            — Cấu hình hạ tầng
├── controller/                        — REST endpoint
├── dto/                               — Request/Response objects
├── entity/                            — JPA entity (bảng DB)
├── exception/                         — Exception + handler
├── repository/                        — Spring Data JPA interface
└── service/                           — Nghiệp vụ + authorization
```

## 8.3. Package `config`

| File | Mục đích |
|---|---|
| `SecurityConfig.java` | Cấu hình Spring Security: stateless, CORS, JWT filter, route permitAll/authenticated, AuthenticationEntryPoint. |
| `JwtUtil.java` | Sinh + verify JWT. Method `generateToken(userId, email)`, `validateToken`, `getEmailFromToken`, `getUserIdFromToken`. |
| `JwtAuthenticationFilter.java` | `OncePerRequestFilter` — đọc `Authorization: Bearer ...`, validate, set SecurityContext với principal là `Long userId`. |
| `JwtAuthenticationEntryPoint.java` | Trả JSON 401 thay HTML login mặc định khi request không có token / token sai. |
| `SecurityUtil.java` | Helper static `currentUserId()` lấy userId từ SecurityContext. |

## 8.4. Package `controller` — 9 REST controller

Mọi controller đều có:
- `@RestController` + `@RequestMapping("/api/...")`
- `@RequiredArgsConstructor` (Lombok inject service)
- Với endpoint theo user/lớp, lấy userId từ `SecurityUtil.currentUserId()` thay vì `@RequestHeader`

| Controller | Endpoint prefix | Số endpoint |
|---|---|---|
| `AuthController` | `/api/auth` | 2 (register, login) |
| `ClassroomController` | `/api/classrooms` | 4 (create, join, my, members) |
| `ClassroomBankAccountController` | `/api/classrooms/{id}/bank-account` | 3 (get, upsert, history) |
| `BankController` | `/api/banks` | 2 (list active banks, sync from VietQR) |
| `FundCollectionController` | `/api/fund` | 8 (collections + payments + mark-paid + QR + status) |
| `FundExpenseController` | `/api/fund/expenses` | 2 (create, list) |
| `EventController` | `/api/events` | 9 (events + detail + volunteer + assign + check-in thủ công) |
| `EventCheckinSubmissionController` | `/api/events/{id}/checkin-submissions`, `/api/events/checkin-submissions/{id}` | 4 (submit, list, approve, reject) |
| `NotificationController` | `/api/notifications` | 4 route mapping (list, unread-count, mark-read, mark-all-read; list/count/read-all hỗ trợ filter `classroomId`) |

Chi tiết: xem `07-api-documentation.md`.

## 8.5. Package `service` — Nghiệp vụ + authorization

| Service | Trách nhiệm |
|---|---|
| `AuthService` | register (check email unique, hash password, sinh JWT). login (verify password, sinh JWT). |
| `ClassroomService` | createClassroom (sinh inviteCode, auto-gán ADMIN). joinClassroom (chống join trùng, **B6: backfill payment cho member join muộn**). getMyClassrooms. |
| `ClassroomBankAccountService` | getBankAccount (lấy tài khoản active). upsertBankAccount validate `bankBin` qua `BankRepository`, snapshot `bankName`/`shortName`, tạo mới hoặc đánh dấu cũ inactive, tạo bản mới active. getHistory (admin-only). |
| `BankService` | getBanks (trả bank active theo `shortName`). syncBanksFromVietQr (gọi `vietqr.banks-url`, validate response code `00`, upsert `banks` theo `bin`, map `transferSupported`/`lookupSupported`). |
| `FundCollectionService` | createCollection (kiểm tra lớp có bank account active, auto-sinh payment cho all members, **gửi notification COLLECTION_CREATED cho member trong lớp, trừ người tạo**). list + get payments. **confirmPayment (B3: lưu confirmedBy + idempotency, gửi notification PAYMENT_CONFIRMED cho chủ payment)**. getMyPayments. **generateQr (lấy bank account active từ DB, owner-only + cache paymentCode)**. getPaymentStatus. |
| `FundExpenseService` | createExpense (requireAdmin). list (requireMember). |
| `EventService` | createEvent với `minParticipants` (requireAdmin, **gửi notification EVENT_CREATED cho member trong lớp, trừ người tạo**). list + getEventDetail + getMyEvents (requireMember). volunteer set `source=VOLUNTEER`. cancelVolunteer chặn `ASSIGNED`, cho `VOLUNTEER` huỷ nếu chưa check-in. getParticipants (requireAdmin). assignParticipants set `source=ASSIGNED`, `assignedBy`, `assignedAt`, distinct `userIds`, skip participant đã tồn tại. **checkIn (B4: lưu checkedBy + idempotency)**. |
| `FileStorageService` | **validateAndStoreFile(MultipartFile)**: kiểm tra contentType bắt đầu `"image/"`, extension thuộc `{jpg,jpeg,png}`, size ≤ 5MB. Lưu file vào `classhub.upload-dir/event-checkins/`, đặt tên `event_{id}_user_{id}_{timestamp}.{ext}`. Trả `imagePath`. |
| `EventCheckinSubmissionService` | **submitCheckin** (requireMember, check participant, chống PENDING trùng, gọi FileStorageService, lưu Submission, gửi `CHECKIN_SUBMITTED` cho admin/lớp trưởng). **getSubmissions** (requireAdmin). **approveSubmission** (requireAdmin, set APPROVED, set checkedIn=true trên participant, gửi `CHECKIN_APPROVED` cho member). **rejectSubmission** (requireAdmin, set REJECTED + lưu lý do, gửi `CHECKIN_REJECTED` cho member). |
| `NotificationService` | **getMyNotifications** (phân trang theo userId, optional `classroomId`, mới nhất trước). **getUnreadCount** (đếm recipient.read=false, optional `classroomId`). **markAsRead** (set read=true + readAt, chỉ chủ recipient). **markAllAsRead** (bulk update unread toàn cục hoặc theo lớp). **createNotification** (internal — tạo Notification + NotificationRecipient cho danh sách userId, dùng `REQUIRES_NEW` transaction). |
| `AuthorizationService` | **Trung tâm phân quyền.** `requireMember(userId, classroomId)` + `requireAdmin(userId, classroomId)`. Vi phạm → `ForbiddenException`. |

### Luồng NotificationService
- `NotificationController` có prefix `/api/notifications`, tất cả endpoint yêu cầu JWT và lấy user bằng `SecurityUtil.currentUserId()`.
- `GET /api/notifications?page=0&size=20` gọi `getMyNotifications(userId, null, pageable)`, query `NotificationRecipientRepository.findByUserIdOrderByCreatedAtDesc(...)`, trả `Page<NotificationResponse>`.
- `GET /api/notifications?classroomId={id}&page=0&size=20` gọi `getMyNotifications(userId, classroomId, pageable)`, query `findByUserIdAndNotification_Classroom_IdOrderByCreatedAtDesc(...)`.
- `GET /api/notifications/unread-count` gọi `countByUserIdAndReadFalse(userId)`, trả `UnreadCountResponse { count }`.
- `GET /api/notifications/unread-count?classroomId={id}` gọi `countByUserIdAndReadFalseAndNotification_Classroom_Id(userId, classroomId)`.
- `PUT /api/notifications/{recipientId}/read` gọi `findByIdAndUserId(recipientId, userId)`. Nếu recipient không thuộc user hiện tại thì trả `ForbiddenException`; nếu chưa đọc thì set `read=true`, `readAt=now`.
- `PUT /api/notifications/read-all` lấy toàn bộ recipient chưa đọc của user hiện tại bằng `findByUserIdAndReadFalse(userId)`, set `read=true`, `readAt=now`, trả `count` là số recipient vừa cập nhật.
- `PUT /api/notifications/read-all?classroomId={id}` lấy recipient chưa đọc bằng `findByUserIdAndReadFalseAndNotification_Classroom_Id(userId, classroomId)`, không ảnh hưởng lớp khác.
- `createNotification(...)` là method internal, nhận `classroomId`, `type`, `title`, `message`, `targetType`, `targetId`, `createdByUserId`, `recipientUserIds`; method lọc `null`, `distinct`, kiểm tra user tồn tại rồi tạo 1 `Notification` và nhiều `NotificationRecipient`.
- `EventService.createEvent(...)` emit `EVENT_CREATED`, `targetType=EVENT`, `targetId=event.id`, gửi cho member trong lớp trừ người tạo.
- `FundCollectionService.createCollection(...)` emit `COLLECTION_CREATED`, `targetType=FUND_COLLECTION`, `targetId=collection.id`, gửi cho member trong lớp trừ người tạo.
- `FundCollectionService.confirmPayment(...)` emit `PAYMENT_CONFIRMED`, `targetType=FUND_PAYMENT`, `targetId=payment.id`, gửi cho member sở hữu payment.
- `EventCheckinSubmissionService.submit(...)` emit `CHECKIN_SUBMITTED`, `targetType=CHECKIN_SUBMISSION`, `targetId=submission.id`, gửi cho admin/lớp trưởng trong lớp.
- `EventCheckinSubmissionService.approve(...)` emit `CHECKIN_APPROVED`, `targetType=CHECKIN_SUBMISSION`, `targetId=submission.id`, gửi cho member chủ submission.
- `EventCheckinSubmissionService.reject(...)` emit `CHECKIN_REJECTED`, `targetType=CHECKIN_SUBMISSION`, `targetId=submission.id`, gửi cho member chủ submission.
- `createNotification(...)` dùng `@Transactional(propagation = Propagation.REQUIRES_NEW)` để tách transaction ghi notification khỏi transaction nghiệp vụ gọi nó. Hiện gọi đồng bộ, chưa có queue/retry riêng.
- Không có endpoint nhận `userId` từ client; filter theo `classroomId` vẫn nằm trong scope current user, không cho đọc/sửa recipient của người khác.

### Luồng BankService
- `GET /api/banks` được `SecurityConfig` mở public (`requestMatchers(HttpMethod.GET, "/api/banks").permitAll()`), trả danh sách bank active theo `findByActiveTrueOrderByShortNameAsc`.
- `POST /api/banks/sync` nằm dưới rule `/api/**.authenticated()`, hiện chỉ yêu cầu JWT vì project chưa có role system-admin/global-admin.
- `syncBanksFromVietQr()` gọi `vietqr.banks-url`, chỉ nhận response có `code="00"` và `data` hợp lệ.
- Upsert dùng `BankRepository.findByBin(...)`; bank mới mặc định `active=true`, bank cũ được cập nhật `vietQrId`, `code`, `name`, `shortName`, `logo`, `transferSupported`, `lookupSupported`.
- MVP limitation: trước production nên restrict `/api/banks/sync` cho system admin hoặc chuyển thành job nội bộ/scheduler.

### Conventions service
- `@Service` + `@RequiredArgsConstructor`
- Method thay đổi state có `@Transactional`
- Mọi action có ràng buộc role/membership **bắt buộc gọi `authorizationService.requireXxx(...)` đầu method**
- Trả về DTO (Response), không trả thẳng entity

## 8.6. Package `repository` — 13 JpaRepository

| Repository | Custom query method quan trọng |
|---|---|
| `UserRepository` | `existsByEmail`, `findByEmail` |
| `ClassroomRepository` | `findByInviteCode` |
| `ClassMemberRepository` | `existsByUserIdAndClassroomId`, `findByUserId`, `findByClassroomId`, `findByUserIdAndClassroomId` (cho B2 authz và API members) |
| `ClassroomBankAccountRepository` | `findByClassroomIdAndActiveTrue`, `findByClassroomIdOrderByCreatedAtDesc` (history) |
| `BankRepository` | `findByBin`, `findByActiveTrueOrderByShortNameAsc` |
| `FundCollectionRepository` | `findByClassroomId`, `existsByIdAndClassroomId` |
| `FundPaymentRepository` | `findByFundCollectionId`, `findByUserIdAndFundCollection_ClassroomId`, `existsByUserIdAndFundCollectionId`, `sumConfirmedAmountByCollectionId` (custom `@Query`) |
| `FundExpenseRepository` | `findByClassroomId` |
| `EventRepository` | `findByClassroomIdOrderByEventTimeDesc` |
| `EventParticipantRepository` | `findByEventId`, `existsByEventIdAndUserId`, `findByEventIdAndUserId`, `findByUserIdAndEvent_ClassroomId`, `countByEventIds` (`@Query` aggregation) |
| `EventCheckinSubmissionRepository` | `findTopByEventIdAndUserIdOrderBySubmittedAtDesc`, `findByEventIdOrderBySubmittedAtDesc`, `existsByEventIdAndUserIdAndStatus` (chống PENDING trùng) |
| `NotificationRepository` | (CRUD cơ bản — Spring Data JPA) |
| `NotificationRecipientRepository` | `findByUserIdOrderByCreatedAtDesc` (phân trang toàn cục), `findByUserIdAndNotification_Classroom_IdOrderByCreatedAtDesc` (phân trang theo lớp), `countByUserIdAndReadFalse`, `countByUserIdAndReadFalseAndNotification_Classroom_Id`, `findByIdAndUserId` (ownership check), `findByUserIdAndReadFalse`, `findByUserIdAndReadFalseAndNotification_Classroom_Id` (bulk mark-read) |

**Conventions:**
- Tận dụng **derived query method** của Spring Data JPA (`findByXxxAndYyy`) — không viết SQL/HQL thủ công.
- Underscore `_` trong tên method để navigate qua FK (vd `findByUserIdAndFundCollection_ClassroomId`).
- Chỉ dùng `@Query` khi cần `SUM`/aggregation.

## 8.7. Package `entity` — 13 JPA entity + enum phụ

| Entity | Bảng | Quan hệ chính |
|---|---|---|
| `User` | `users` | (root) |
| `Classroom` | `classrooms` | `createdBy` (Long, chưa @ManyToOne) |
| `ClassMember` | `class_members` | `@ManyToOne User`, `@ManyToOne Classroom`, enum `Role{ADMIN,MEMBER}` |
| `Bank` | `banks` | Danh mục VietQR: `bin` unique, `code`, `name`, `shortName`, `logo`, `transferSupported`, `lookupSupported`, `active` |
| `ClassroomBankAccount` | `classroom_bank_accounts` | `@ManyToOne Classroom`, `@ManyToOne User createdBy`, `bankBin`, snapshot `bankName`/`shortName`, boolean `active` |
| `FundCollection` | `fund_collections` | `@ManyToOne Classroom`, `@ManyToOne User createdBy` |
| `FundPayment` | `fund_payments` | `@ManyToOne User user`, `@ManyToOne FundCollection`, **`@ManyToOne User confirmedBy` (B3)** |
| `FundExpense` | `fund_expenses` | `@ManyToOne Classroom`, `@ManyToOne User createdBy` |
| `Event` | `events` | `@ManyToOne Classroom`, `@ManyToOne User createdBy`, `minParticipants` |
| `EventParticipant` | `event_participants` | `@ManyToOne Event`, `@ManyToOne User user`, **`@ManyToOne User checkedBy` (B4)**, enum `source`, `@ManyToOne User assignedBy`, `assignedAt` |
| `EventCheckinSubmission` | `event_checkin_submissions` | `@ManyToOne Event`, `@ManyToOne User user`, `@ManyToOne EventParticipant participant`, **`@ManyToOne User reviewedBy`**, enum `status {PENDING, APPROVED, REJECTED}`, `imagePath`, `rejectedReason` |
| `Notification` | `notifications` | `@ManyToOne Classroom`, `@ManyToOne User createdBy`, enum `type`, enum `targetType`, `targetId` (FK ngầm đến đối tượng liên quan) |
| `NotificationRecipient` | `notification_recipients` | `@ManyToOne Notification`, `@ManyToOne User`, boolean `read`, `readAt`. UNIQUE(notification_id, user_id). |

**Enum phụ:**
| Enum | Giá trị |
|---|---|
| `NotificationType` | `COLLECTION_CREATED`, `PAYMENT_CONFIRMED`, `EVENT_CREATED`, `CHECKIN_SUBMITTED`, `CHECKIN_APPROVED`, `CHECKIN_REJECTED` |
| `NotificationTargetType` | `CLASSROOM`, `FUND_COLLECTION`, `FUND_PAYMENT`, `EVENT`, `CHECKIN_SUBMISSION` |
| `ParticipantSource` | `VOLUNTEER`, `ASSIGNED` |

**Conventions:**
- `@Entity` + `@Table(name="...")`
- `@Id @GeneratedValue(strategy=IDENTITY)` cho khoá chính
- Lombok: `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`
- `@CreationTimestamp` + `LocalDateTime createdAt` (auto)
- Mọi `@ManyToOne` đều `fetch=FetchType.LAZY`
- `@UniqueConstraint` cho cặp key chống trùng

## 8.8. Package `dto`

DTO chia 2 loại + DTO Camera Check-in + DTO Notification:

| Loại | File |
|---|---|
| Request | `RegisterRequest`, `LoginRequest`, `CreateClassroomRequest`, `JoinClassroomRequest`, `UpdateClassroomBankAccountRequest` (`bankBin`, `accountNo`, `accountName`, `note`), `CreateCollectionRequest`, `CreateExpenseRequest`, `CreateEventRequest` (`minParticipants`), `AssignEventParticipantsRequest`, `RejectCheckinSubmissionRequest` |
| Response | `AuthResponse`, `ClassroomResponse`, `ClassMemberResponse`, `BankResponse`, `VietQrBanksResponse`, `ClassroomBankAccountResponse` (`shortName`), `CollectionResponse`, `PaymentResponse`, `PaymentStatusResponse`, `QrResponse`, `ExpenseResponse`, `EventResponse` (`minParticipants`), `EventParticipantResponse` (`source`, `assignedByName`, `assignedAt`), `EventCheckinSubmissionResponse`, `NotificationResponse`, `UnreadCountResponse` |

**`BankResponse` fields:** `id`, `bankBin`, `code`, `name`, `shortName`, `logo`, `transferSupported`, `lookupSupported`. Response dùng `bankBin`, không dùng `bin`.

**`VietQrBanksResponse` fields:** `code`, `desc`, `data[]`; mỗi item gồm `id`, `name`, `code`, `bin`, `shortName`, `logo`, `transferSupported`, `lookupSupported`.

**`AssignEventParticipantsRequest` fields:** `userIds: List<Long>`; validate không rỗng ở service, duplicate được distinct.

**`EventParticipantResponse` fields mới:** `source`, `assignedByName`, `assignedAt`. Nếu data cũ `source == null`, mapper/service fallback thành `VOLUNTEER`.

**`NotificationResponse` fields:** `recipientId`, `notificationId`, `classroomId`, `type` (enum), `title`, `message`, `targetType` (enum), `targetId`, `isRead`, `readAt`, `createdAt`, `createdByName`.

**`UnreadCountResponse` fields:** `count` (long).

**Conventions:**
- Request có `@NotBlank`/`@NotNull`/`@Email`/`@DecimalMin` cho validation.
- Response có `@Builder` để code đẹp `XxxResponse.builder().a(..).b(..).build()`.
- Không expose entity ra ngoài; mọi conversion entity → DTO làm trong service.

## 8.9. Package `exception`

| File | Mục đích |
|---|---|
| `BadRequestException` | RuntimeException → 400. Dùng khi validate nghiệp vụ fail (vd "email đã tồn tại"). |
| `ForbiddenException` | RuntimeException → 403. Dùng khi user không có quyền (vd không phải admin lớp). |
| `GlobalExceptionHandler` | `@RestControllerAdvice` cover 6 loại exception: BadRequest, Forbidden, MethodArgumentNotValid, MissingRequestHeader, HttpMessageNotReadable, generic Exception. Mọi response đều JSON. |

## 8.10. File cấu hình

### `application.properties`
```properties
# DB
spring.datasource.url=jdbc:mysql://localhost:3306/classhub_db?...
spring.datasource.username=classhub_user
spring.datasource.password=ClassHub@2026

# JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false                # Tắt SQL logging để không log số tài khoản

# Server
server.port=8080

# JWT
jwt.secret=classhub-super-secret-key-2026-do-an-tot-nghiep-spring-boot-flutter
jwt.expiration=86400000   # 24h

# VietQR
vietqr.template=compact2
vietqr.banks-url=https://api.vietqr.io/v2/banks
```

### `pom.xml` — dependency chính
- spring-boot-starter-data-jpa
- spring-boot-starter-security
- spring-boot-starter-validation
- spring-boot-starter-webmvc
- mysql-connector-j
- lombok
- jjwt-api / jjwt-impl / jjwt-jackson (0.12.6)

## 8.11. Luồng khởi động Spring Boot

1. **`ClasshubApiApplication.main()`** chạy `SpringApplication.run(...)`.
2. Spring scan `@Configuration` classes → load `SecurityConfig`.
3. Spring scan `@Entity` → Hibernate đọc → so sánh với schema MySQL → `ddl-auto=update` tự `ALTER TABLE` thêm cột mới.
4. Tomcat embedded khởi động tại port `8080`.
5. Filter chain:
   ```
   Request → CORS filter → JwtAuthenticationFilter →
   Spring Security authentication check → Controller
   ```
6. Khi có exception → `GlobalExceptionHandler` catch → trả JSON.

## 8.12. Luồng xử lý một request (ví dụ `PUT /payments/123/confirm`)

```
1. CORS preflight (nếu OPTIONS) → CorsConfigurationSource cho phép

2. JwtAuthenticationFilter:
   - Đọc header Authorization
   - jwtUtil.validateToken(token)  → true
   - jwtUtil.getUserIdFromToken(token) → 7
   - SecurityContextHolder.setAuthentication(token với principal=7L)

3. Spring Security authorize:
   - Route /api/** yêu cầu authenticated()
   - SecurityContext có authentication → pass

4. DispatcherServlet route đến FundCollectionController.confirm(123)

5. Controller:
   - SecurityUtil.currentUserId() → 7
   - Gọi fundCollectionService.confirmPayment(123, 7)

6. Service @Transactional:
   - Load FundPayment id=123
   - classroomId = payment.fundCollection.classroom.id (lazy load 2 lần)
   - authorizationService.requireAdmin(7, classroomId)
     → Query class_members WHERE user_id=7 AND classroom_id=...
     → Role=ADMIN ✅
   - Check payment.confirmedByAdmin == false ✅
   - Load admin User id=7
   - payment.setConfirmedBy(admin), setPaid(true), setConfirmedByAdmin(true), setPaidAt(now)
   - fundPaymentRepository.save(payment)
     → SQL: UPDATE fund_payments SET ... WHERE id=123
   - Convert → PaymentResponse với confirmedByName

7. Controller trả ResponseEntity.ok(response)

8. Spring serialize → JSON response body
```

## 8.13. Đặc điểm code

| Đặc điểm | Cụ thể |
|---|---|
| **Layered** | controller → service → repository → entity, không skip tầng |
| **Stateless** | JWT, không session HttpSession |
| **Transactional** | Mọi action thay đổi state: `@Transactional` |
| **Validation 3 lớp** | (1) DTO `@Valid`, (2) service business check, (3) DB constraint |
| **Lazy loading** | `@ManyToOne(fetch=LAZY)` để tránh load không cần thiết |
| **DRY authorization** | `AuthorizationService` gom 2 method dùng ở mọi service |
| **Audit trail** | Lưu `confirmedBy` + `paidAt` (B3); `checkedBy` + `checkedInAt` (B4); `assignedBy` + `assignedAt` cho participant được BCS thêm |
| **Idempotency** | confirm 2 lần + check-in 2 lần đều bị chặn; assign participant distinct `userIds` và skip user đã tham gia |

## 8.14. Trạng thái compile

```
$ ./mvnw clean compile
[INFO] BUILD SUCCESS

$ ./mvnw test
[INFO] Tests run: 1, Failures: 0, Errors: 0
[INFO] BUILD SUCCESS
```

1 warning deprecated `WebAuthenticationDetailsSource` ở `JwtAuthenticationFilter` — vô hại.

## 8.15. Phần BE còn lại (chưa làm)

Xem chi tiết: `classhub-api/documents/PROJECT_STATUS.md`. Tóm tắt:

| Mức | Việc |
|---|---|
| 🔴 Phải làm trước demo | API thống kê quỹ; FE tích hợp Event detail/assign nếu muốn demo luồng chỉ định participant |
| 🟡 Nên có | `@Future` cho deadline/eventTime; `Event.endTime`; API edit/delete; Test case tự động JUnit/MockMvc |
| 🟢 Hướng phát triển | Role OWNER, attendance status chi tiết hơn cho participant, notification push (Firebase FCM), reminder/scheduler, user notification settings, device token, delivery logs, deep link chi tiết từ notification, refresh token, đối soát ngân hàng |

> **Notification in-app đã implement** (polling từ FE). Notification push qua Firebase FCM là hướng mở rộng sau.

## 8.16. Trả lời câu hỏi điển hình

| Câu hỏi | Trả lời |
|---|---|
| "JWT lưu ở đâu?" | Sinh ở `AuthService.login`, trả về client. Client lưu `SharedPreferences['jwt_token']`. Mỗi request gắn `Authorization: Bearer`. Server stateless, không lưu. |
| "Password mã hoá kiểu gì?" | BCrypt với cost 10 (mặc định Spring). Salt nhúng trong hash. |
| "Phân quyền chỗ nào?" | Trong từng service, gọi `AuthorizationService.requireMember/requireAdmin` đầu method. Filter chỉ authenticate, không authorize. |
| "Tại sao role không nằm ở User?" | Vì 1 user có thể tham gia nhiều lớp với role khác nhau. Role thuộc cặp (user, classroom) → đặt ở `ClassMember`. |
| "DB đổi thì sao?" | Đổi `application.properties` (URL, user, password). Hibernate `ddl-auto=update` tự sinh schema. |
| "Test thế nào?" | Hiện BE Event mới đã chạy `mvnw.cmd clean compile` BUILD SUCCESS và `mvnw.cmd test` pass (1 test, 0 failures/errors). Test case đặc tả nằm trong `10-kiem-thu.md`, sẽ implement bằng MockMvc trong giai đoạn tiếp theo. |
| "File ảnh lưu ở đâu?" | `D:/big_dream/classhub-uploads/event-checkins/` (ngoài repo). Cấu hình bằng `classhub.upload-dir` trong `application.properties`. DB chỉ lưu `image_path` và metadata. |
| "`/uploads/**` có bảo mật không?" | Hiện public cho MVP/demo (`WebMvcConfig.addResourceHandlers`). Hướng phát triển: API download có JWT. |
| "Notification hoạt động kiểu gì?" | In-app notification dùng polling (FE gọi `GET /api/notifications/unread-count` toàn cục ở `HomeScreen`, gọi thêm biến thể `?classroomId=` trong `ClassroomDetailScreen`, và load inbox ở `NotificationScreen`). Service nghiệp vụ gọi `NotificationService.createNotification()` để sinh notification khi tạo khoản thu, xác nhận thanh toán, tạo sự kiện, gửi ảnh check-in, duyệt ảnh, hoặc từ chối ảnh. Method này dùng `REQUIRES_NEW` propagation để tách transaction ghi notification. Không dùng WebSocket, Firebase FCM hay push notification ngoài app ở giai đoạn hiện tại; tap notification chưa deep-link chi tiết theo tab/bản ghi. |
