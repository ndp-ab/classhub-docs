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

## 8.4. Package `controller` — 5 REST controller

Mọi controller đều có:
- `@RestController` + `@RequestMapping("/api/...")`
- `@RequiredArgsConstructor` (Lombok inject service)
- Lấy userId từ `SecurityUtil.currentUserId()` thay vì `@RequestHeader`

| Controller | Endpoint prefix | Số endpoint |
|---|---|---|
| `AuthController` | `/api/auth` | 2 (register, login) |
| `ClassroomController` | `/api/classrooms` | 3 (create, join, my) |
| `FundCollectionController` | `/api/fund` | 7 (collections + payments + QR + status) |
| `FundExpenseController` | `/api/fund/expenses` | 2 (create, list) |
| `EventController` | `/api/events` | 7 (events + volunteer + check-in) |

Chi tiết: xem `07-api-documentation.md`.

## 8.5. Package `service` — Nghiệp vụ + authorization

| Service | Trách nhiệm |
|---|---|
| `AuthService` | register (check email unique, hash password, sinh JWT). login (verify password, sinh JWT). |
| `ClassroomService` | createClassroom (sinh inviteCode, auto-gán ADMIN). joinClassroom (chống join trùng, **B6: backfill payment cho member join muộn**). getMyClassrooms. |
| `FundCollectionService` | createCollection (auto-sinh payment cho all members). list + get payments. **confirmPayment (B3: lưu confirmedBy + idempotency)**. getMyPayments. generateQr (owner-only + cache paymentCode). getPaymentStatus. |
| `FundExpenseService` | createExpense (requireAdmin). list (requireMember). |
| `EventService` | createEvent (requireAdmin). list + getMyEvents (requireMember). volunteer + cancelVolunteer (chống đăng ký trùng, chặn huỷ sau check-in). getParticipants (requireAdmin). **checkIn (B4: lưu checkedBy + idempotency)**. |
| `AuthorizationService` | **Trung tâm phân quyền.** `requireMember(userId, classroomId)` + `requireAdmin(userId, classroomId)`. Vi phạm → `ForbiddenException`. |

### Conventions service
- `@Service` + `@RequiredArgsConstructor`
- Method thay đổi state có `@Transactional`
- Mọi action có ràng buộc role/membership **bắt buộc gọi `authorizationService.requireXxx(...)` đầu method**
- Trả về DTO (Response), không trả thẳng entity

## 8.6. Package `repository` — 8 JpaRepository

| Repository | Custom query method quan trọng |
|---|---|
| `UserRepository` | `existsByEmail`, `findByEmail` |
| `ClassroomRepository` | `findByInviteCode`, `existsByIdAndClassroomId` |
| `ClassMemberRepository` | `existsByUserIdAndClassroomId`, `findByUserId`, `findByClassroomId`, `findByUserIdAndClassroomId` (cho B2 authz) |
| `FundCollectionRepository` | `findByClassroomId`, `existsByIdAndClassroomId` |
| `FundPaymentRepository` | `findByFundCollectionId`, `findByUserIdAndFundCollection_ClassroomId`, `existsByUserIdAndFundCollectionId`, `sumConfirmedAmountByCollectionId` (custom `@Query`) |
| `FundExpenseRepository` | `findByClassroomId` |
| `EventRepository` | `findByClassroomIdOrderByEventTimeDesc` |
| `EventParticipantRepository` | `findByEventId`, `existsByEventIdAndUserId`, `findByEventIdAndUserId`, `findByUserIdAndEvent_ClassroomId` |

**Conventions:**
- Tận dụng **derived query method** của Spring Data JPA (`findByXxxAndYyy`) — không viết SQL/HQL thủ công.
- Underscore `_` trong tên method để navigate qua FK (vd `findByUserIdAndFundCollection_ClassroomId`).
- Chỉ dùng `@Query` khi cần `SUM`/aggregation.

## 8.7. Package `entity` — 8 JPA entity

| Entity | Bảng | Quan hệ chính |
|---|---|---|
| `User` | `users` | (root) |
| `Classroom` | `classrooms` | `createdBy` (Long, chưa @ManyToOne) |
| `ClassMember` | `class_members` | `@ManyToOne User`, `@ManyToOne Classroom`, enum `Role{ADMIN,MEMBER}` |
| `FundCollection` | `fund_collections` | `@ManyToOne Classroom`, `@ManyToOne User createdBy` |
| `FundPayment` | `fund_payments` | `@ManyToOne User user`, `@ManyToOne FundCollection`, **`@ManyToOne User confirmedBy` (B3)** |
| `FundExpense` | `fund_expenses` | `@ManyToOne Classroom`, `@ManyToOne User createdBy` |
| `Event` | `events` | `@ManyToOne Classroom`, `@ManyToOne User createdBy` |
| `EventParticipant` | `event_participants` | `@ManyToOne Event`, `@ManyToOne User user`, **`@ManyToOne User checkedBy` (B4)** |

**Conventions:**
- `@Entity` + `@Table(name="...")`
- `@Id @GeneratedValue(strategy=IDENTITY)` cho khoá chính
- Lombok: `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`
- `@CreationTimestamp` + `LocalDateTime createdAt` (auto)
- Mọi `@ManyToOne` đều `fetch=FetchType.LAZY`
- `@UniqueConstraint` cho cặp key chống trùng

## 8.8. Package `dto`

16 DTO chia 2 loại:

| Loại | File |
|---|---|
| Request | `RegisterRequest`, `LoginRequest`, `CreateClassroomRequest`, `JoinClassroomRequest`, `CreateCollectionRequest`, `CreateExpenseRequest`, `CreateEventRequest` |
| Response | `AuthResponse`, `ClassroomResponse`, `CollectionResponse`, `PaymentResponse`, `PaymentStatusResponse`, `QrResponse`, `ExpenseResponse`, `EventResponse`, `EventParticipantResponse` |

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
spring.jpa.show-sql=true

# Server
server.port=8080

# JWT
jwt.secret=classhub-super-secret-key-2026-do-an-tot-nghiep-spring-boot-flutter
jwt.expiration=86400000   # 24h

# VietQR — Cần đổi TK thật trước demo
vietqr.bank-bin=970415
vietqr.account-no=109875610620
vietqr.account-name=Nguyen Duy Phong
vietqr.template=compact2
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
| **Audit trail** | Lưu `confirmedBy` + `paidAt` (B3); `checkedBy` + `checkedInAt` (B4) |
| **Idempotency** | confirm 2 lần + check-in 2 lần đều bị chặn |

## 8.14. Trạng thái compile

```
$ ./mvnw clean compile
[INFO] Compiling 52 source files
[INFO] BUILD SUCCESS
```

1 warning deprecated `WebAuthenticationDetailsSource` ở `JwtAuthenticationFilter` — vô hại.

## 8.15. Phần BE còn lại (chưa làm)

Xem chi tiết: `classhub-api/documents/PROJECT_STATUS.md`. Tóm tắt:

| Mức | Việc |
|---|---|
| 🔴 Phải làm trước demo | API `/classrooms/{id}/members`, API thống kê quỹ |
| 🟡 Nên có | `@Future` cho deadline/eventTime; `Event.endTime`; API edit/delete; Test case TC01–TC20 |
| 🟢 Hướng phát triển | Role OWNER, `EventParticipant.type`/`attendanceStatus`, notification, refresh token, đối soát ngân hàng |

## 8.16. Trả lời câu hỏi điển hình

| Câu hỏi | Trả lời |
|---|---|
| "JWT lưu ở đâu?" | Sinh ở `AuthService.login`, trả về client. Client lưu `SharedPreferences['jwt_token']`. Mỗi request gắn `Authorization: Bearer`. Server stateless, không lưu. |
| "Password mã hoá kiểu gì?" | BCrypt với cost 10 (mặc định Spring). Salt nhúng trong hash. |
| "Phân quyền chỗ nào?" | Trong từng service, gọi `AuthorizationService.requireMember/requireAdmin` đầu method. Filter chỉ authenticate, không authorize. |
| "Tại sao role không nằm ở User?" | Vì 1 user có thể tham gia nhiều lớp với role khác nhau. Role thuộc cặp (user, classroom) → đặt ở `ClassMember`. |
| "DB đổi thì sao?" | Đổi `application.properties` (URL, user, password). Hibernate `ddl-auto=update` tự sinh schema. |
| "Test thế nào?" | Hiện có file `ClasshubApiApplicationTests` rỗng. Test case TC01–TC20 nằm trong `10-kiem-thu.md`, sẽ implement bằng MockMvc trong giai đoạn tiếp theo. |
