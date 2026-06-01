# 05 — Sequence Diagram

Tài liệu này mô tả 6 luồng nghiệp vụ chính bằng **Mermaid sequence diagram** (render được trên GitHub / VS Code preview).

## 5.1. Đăng nhập

```mermaid
sequenceDiagram
    actor U as User
    participant F as Flutter App
    participant LC as AuthController
    participant LS as AuthService
    participant UR as UserRepository
    participant JU as JwtUtil
    participant DB as MySQL

    U->>F: Nhập email + password, bấm "Đăng nhập"
    F->>LC: POST /api/auth/login {email, password}
    LC->>LS: login(request)
    LS->>UR: findByEmail(email)
    UR->>DB: SELECT * FROM users WHERE email=?
    DB-->>UR: User row
    UR-->>LS: User entity
    LS->>LS: passwordEncoder.matches(plain, hash)?

    alt password OK
        LS->>JU: generateToken(userId, email)
        JU-->>LS: JWT string
        LS-->>LC: AuthResponse(token, userId, fullName, email)
        LC-->>F: 200 OK + body
        F->>F: SharedPreferences.setString('jwt_token', token)
        F->>U: Mở HomeScreen
    else password sai
        LS-->>LC: BadRequestException("Mật khẩu không đúng")
        LC-->>F: 400 {message: "Mật khẩu không đúng"}
        F->>U: Hiện SnackBar đỏ
    end
```

## 5.2. Admin tạo khoản thu (auto-sinh payment cho all members)

```mermaid
sequenceDiagram
    actor A as Admin
    participant F as Flutter
    participant JF as JwtAuthFilter
    participant FC as FundCollectionController
    participant FS as FundCollectionService
    participant AS as AuthorizationService
    participant CMR as ClassMemberRepo
    participant FCR as FundCollectionRepo
    participant FPR as FundPaymentRepo
    participant DB as MySQL

    A->>F: Nhập title, amount, deadline, bấm "Tạo"
    F->>JF: POST /api/fund/collections (Bearer token)
    JF->>JF: validateToken(jwt)
    JF->>JF: Set SecurityContext (userId=adminId)
    JF->>FC: forward request
    FC->>FS: createCollection(request, adminId)

    FS->>AS: requireAdmin(adminId, classroomId)
    AS->>CMR: findByUserIdAndClassroomId(adminId, classroomId)
    CMR-->>AS: ClassMember{role=ADMIN}
    AS-->>FS: ✅ OK

    FS->>FCR: save(new FundCollection)
    FCR->>DB: INSERT INTO fund_collections
    DB-->>FCR: id=42

    FS->>CMR: findByClassroomId(classroomId)
    CMR-->>FS: List<ClassMember> (vd 30 thành viên)

    loop For each member
        FS->>FPR: existsByUserIdAndFundCollectionId(userId, 42)
        FPR-->>FS: false
        FS->>FPR: save(new FundPayment(user, collection, false))
        FPR->>DB: INSERT INTO fund_payments
    end

    FS-->>FC: CollectionResponse{id:42, totalMembers:30, paidCount:0}
    FC-->>F: 200 OK
    F->>A: Reload danh sách, đợt thu mới hiện ra
```

## 5.3. Admin xác nhận thanh toán (B3: lưu confirmedBy + idempotency)

```mermaid
sequenceDiagram
    actor A as Admin
    participant F as Flutter
    participant FC as FundCollectionController
    participant FS as FundCollectionService
    participant AS as AuthorizationService
    participant FPR as FundPaymentRepo
    participant UR as UserRepository
    participant DB as MySQL

    A->>F: Bấm "Xác nhận" cho SV B
    F->>F: Show confirm dialog
    A->>F: Bấm "Xác nhận" lần 2
    F->>FC: PUT /api/fund/payments/{id}/confirm (Bearer)

    FC->>FS: confirmPayment(paymentId, adminId)
    FS->>FPR: findById(paymentId)
    FPR->>DB: SELECT
    DB-->>FPR: payment (user=B, collection.classroom.id=classroomId)
    FPR-->>FS: payment

    FS->>AS: requireAdmin(adminId, classroomId)
    AS-->>FS: ✅ OK

    alt payment.confirmedByAdmin == true
        FS-->>FC: BadRequestException("Đã xác nhận rồi")
        FC-->>F: 400
    else chưa xác nhận
        FS->>UR: findById(adminId)
        UR-->>FS: admin User
        FS->>FS: payment.setPaid(true)
        FS->>FS: payment.setConfirmedByAdmin(true)
        FS->>FS: payment.setPaidAt(now)
        FS->>FS: payment.setConfirmedBy(admin)
        FS->>FPR: save(payment)
        FPR->>DB: UPDATE fund_payments
        FS-->>FC: PaymentResponse{confirmedByName:"Admin A"}
        FC-->>F: 200 OK
        F->>A: SnackBar "Đã xác nhận"
        F->>F: Reload danh sách
    end
```

## 5.4. Sinh viên lấy QR thanh toán + polling

```mermaid
sequenceDiagram
    actor M as Member
    participant F as Flutter
    participant FC as FundCollectionController
    participant FS as FundCollectionService
    participant FPR as FundPaymentRepo
    participant VietQR as img.vietqr.io
    participant DB as MySQL

    M->>F: Bấm "Xem QR" trên khoản chưa đóng

    Note over F,FC: Lần 1 — lấy QR
    F->>FC: GET /api/fund/payments/{id}/qr (Bearer)
    FC->>FS: generateQr(paymentId, userId)
    FS->>FPR: findById(paymentId)
    FPR->>DB: SELECT
    DB-->>FPR: payment

    alt payment.user.id != userId
        FS-->>FC: ForbiddenException
        FC-->>F: 403
    else đúng chủ payment
        alt payment.paymentCode == null
            FS->>FS: Sinh code "QUY42-SV7-1715..."
            FS->>FPR: save(payment with code)
        end

        FS->>FS: Build VietQR URL
        FS-->>FC: QrResponse{qrUrl, amount, paymentCode}
        FC-->>F: 200 OK
        F->>VietQR: GET qrUrl
        VietQR-->>F: PNG image
        F->>M: Hiển thị QR + nội dung CK
    end

    Note over F,FC: Polling mỗi 5s
    loop Mỗi 5 giây cho đến khi CONFIRMED
        F->>FC: GET /api/fund/payments/{id}/status
        FC->>FS: getPaymentStatus(paymentId, userId)
        FS->>FPR: findById(paymentId)
        FPR-->>FS: payment

        alt confirmedByAdmin == true
            FS-->>FC: status="CONFIRMED"
            FC-->>F: 200 OK
            F->>F: Cancel timer
            F->>M: Hiển thị "Đã thanh toán & được xác nhận"
        else chưa
            FS-->>FC: status="PENDING"
            FC-->>F: 200 OK
        end
    end
```

## 5.5. Sinh viên đăng ký tham gia sự kiện

```mermaid
sequenceDiagram
    actor M as Member
    participant F as Flutter
    participant EC as EventController
    participant ES as EventService
    participant AS as AuthorizationService
    participant ER as EventRepository
    participant EPR as EventParticipantRepo
    participant DB as MySQL

    M->>F: Bấm "Đăng ký" trên sự kiện X
    F->>EC: POST /api/events/{eventId}/volunteer (Bearer)

    EC->>ES: volunteer(eventId, userId)
    ES->>ER: findById(eventId)
    ER->>DB: SELECT event JOIN classroom
    DB-->>ER: Event entity
    ER-->>ES: event

    ES->>AS: requireMember(userId, event.classroom.id)
    AS-->>ES: ✅ OK (user thuộc lớp)

    ES->>EPR: existsByEventIdAndUserId(eventId, userId)
    EPR-->>ES: false

    ES->>EPR: save(new EventParticipant{event, user, checkedIn=false})
    EPR->>DB: INSERT INTO event_participants
    Note over DB: UNIQUE(event_id, user_id) đảm bảo không trùng

    ES-->>EC: EventParticipantResponse
    EC-->>F: 200 OK
    F->>F: Reload events
    F->>M: Chip "Đã đăng ký" hiển thị
```

## 5.6. Admin check-in sự kiện (B4: lưu checkedBy)

```mermaid
sequenceDiagram
    actor A as Admin
    participant F as Flutter
    participant EC as EventController
    participant ES as EventService
    participant AS as AuthorizationService
    participant ER as EventRepo
    participant EPR as EventParticipantRepo
    participant UR as UserRepo
    participant DB as MySQL

    A->>F: Bấm "Check-in" cho SV B
    F->>F: Show confirm dialog
    A->>F: Bấm "Check-in" lần 2

    F->>EC: PUT /api/events/{eventId}/checkin/{userId} (Bearer)
    EC->>ES: checkIn(eventId, targetUserId, adminId)

    ES->>ER: findById(eventId)
    ER-->>ES: event

    ES->>AS: requireAdmin(adminId, event.classroom.id)
    AS-->>ES: ✅ OK

    ES->>EPR: findByEventIdAndUserId(eventId, targetUserId)
    EPR-->>ES: participant

    alt participant.checkedIn == true
        ES-->>EC: BadRequestException("Đã check-in rồi")
        EC-->>F: 400
    else chưa check-in
        ES->>UR: findById(adminId)
        UR-->>ES: admin

        ES->>ES: participant.setCheckedIn(true)
        ES->>ES: participant.setCheckedInAt(now)
        ES->>ES: participant.setCheckedBy(admin)
        ES->>EPR: save(participant)
        EPR->>DB: UPDATE event_participants

        ES-->>EC: EventParticipantResponse{checkedByName:"Admin A"}
        EC-->>F: 200 OK
        F->>A: SnackBar "Đã check-in"
        F->>F: Reload
    end
```

## 5.7. Tổng kết

| Sequence | Tính nghiệp vụ chính được thể hiện |
|---|---|
| 5.1 Đăng nhập | Password hash + JWT generation |
| 5.2 Tạo khoản thu | Auto-sinh payment cho all members; requireAdmin |
| 5.3 Xác nhận thanh toán | Idempotency check; lưu confirmedBy |
| 5.4 QR + polling | Owner-only check; 5s polling; dispose timer |
| 5.5 Volunteer | requireMember; chống đăng ký trùng |
| 5.6 Check-in | requireAdmin; idempotency; lưu checkedBy |

**Điểm chung của mọi sequence:**
- Mọi request (trừ /auth) đều đi qua `JwtAuthenticationFilter`.
- Service không tin user thông tin từ client — luôn check `AuthorizationService.requireMember/requireAdmin` trước action.
- Mọi action thay đổi state đều `@Transactional` để rollback nếu fail giữa chừng.
