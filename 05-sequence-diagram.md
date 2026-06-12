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

    ES->>EPR: save(new EventParticipant{event, user, checkedIn=false, source=VOLUNTEER})
    EPR->>DB: INSERT INTO event_participants
    Note over DB: UNIQUE(event_id, user_id) đảm bảo không trùng

    ES-->>EC: EventParticipantResponse
    EC-->>F: 200 OK
    F->>F: Reload events
    F->>M: Chip "Đã đăng ký" hiển thị
```

## 5.6. Admin bổ sung người tham gia khi chưa đủ tối thiểu

```mermaid
sequenceDiagram
    actor A as Admin
    actor M as Member
    participant F as Flutter
    participant EC as EventController
    participant CC as ClassroomController
    participant ES as EventService
    participant AS as AuthorizationService
    participant ER as EventRepo
    participant CMR as ClassMemberRepo
    participant EPR as EventParticipantRepo
    participant UR as UserRepo
    participant DB as MySQL

    A->>F: Tạo sự kiện, nhập minParticipants
    F->>EC: POST /api/events {classroomId, title, eventTime, minParticipants}
    EC->>ES: createEvent(request, adminId)
    ES->>AS: requireAdmin(adminId, classroomId)
    ES->>ER: save(Event{minParticipants})
    ER->>DB: INSERT INTO events

    M->>F: Tự đăng ký tham gia
    F->>EC: POST /api/events/{eventId}/volunteer
    EC->>ES: volunteer(eventId, memberId)
    ES->>EPR: save(EventParticipant{source=VOLUNTEER})
    EPR->>DB: INSERT INTO event_participants

    A->>F: Mở chi tiết sự kiện
    F->>EC: GET /api/events/detail/{eventId}
    EC->>ES: getEventDetail(eventId, currentUserId)
    ES->>AS: requireMember(currentUserId, event.classroom.id)
    ES->>EPR: countByEventId(eventId)
    ES-->>EC: EventResponse{minParticipants, volunteerCount}
    EC-->>F: 200 OK
    F->>A: Hiển thị "Đã tham gia X/Y, còn thiếu N"

    alt chưa đủ người
        F->>CC: GET /api/classrooms/{classroomId}/members
        CC-->>F: Danh sách thành viên lớp
        A->>F: Chọn member cần thêm
        F->>EC: POST /api/events/{eventId}/participants/assign {userIds}
        EC->>ES: assignParticipants(eventId, userIds, adminId)
        ES->>AS: requireAdmin(adminId, event.classroom.id)
        ES->>ES: distinct(userIds)
        ES->>UR: findAllById(userIds)
        ES->>CMR: validate users thuộc lớp event
        ES->>EPR: find existing participants
        loop từng user hợp lệ chưa tham gia
            ES->>EPR: save(EventParticipant{source=ASSIGNED, assignedBy=admin, assignedAt=now})
            EPR->>DB: INSERT INTO event_participants
        end
        Note over ES,EPR: User đã tham gia thì skip; VOLUNTEER không đổi thành ASSIGNED
        ES-->>EC: List<EventParticipantResponse>
        EC-->>F: 200 OK
        F->>A: Reload detail/participants
    end
```

## 5.7. Sinh viên huỷ tham gia sự kiện

```mermaid
sequenceDiagram
    actor M as Member
    participant F as Flutter
    participant EC as EventController
    participant ES as EventService
    participant AS as AuthorizationService
    participant EPR as EventParticipantRepo
    participant DB as MySQL

    M->>F: Bấm "Huỷ tham gia"
    F->>EC: DELETE /api/events/{eventId}/volunteer (Bearer)
    EC->>ES: cancelVolunteer(eventId, userId)
    ES->>AS: requireMember(userId, event.classroom.id)
    ES->>EPR: findByEventIdAndUserId(eventId, userId)
    EPR-->>ES: participant

    alt participant.source == ASSIGNED
        ES-->>EC: BadRequestException("Người được BCS thêm không thể tự huỷ")
        EC-->>F: 400
    else participant.source == VOLUNTEER or null fallback VOLUNTEER
        alt participant.checkedIn == true
            ES-->>EC: BadRequestException("Không thể hủy đăng ký sau khi đã check-in")
            EC-->>F: 400
        else chưa check-in
            ES->>EPR: delete(participant)
            EPR->>DB: DELETE FROM event_participants
            EC-->>F: 204 No Content
            F->>M: Chip quay về "Đăng ký"
        end
    end
```

## 5.8. Admin check-in sự kiện (B4: lưu checkedBy)

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

## 5.9. Camera Check-in: Member gửi ảnh minh chứng

```mermaid
sequenceDiagram
    actor M as Member
    participant F as Flutter
    participant EC as EventCheckinSubmissionController
    participant ES as EventCheckinSubmissionService
    participant FS as FileStorageService
    participant EPR as EventParticipantRepo
    participant ECSR as EventCheckinSubmissionRepo
    participant DB as MySQL
    participant Disk as FileSystem

    M->>F: Bấm "Chụp ảnh điểm danh"
    F->>F: ImagePicker.getImage(source: camera, quality: 80)
    F->>F: Hiển thị preview ảnh
    M->>F: Bấm "Gửi minh chứng"

    F->>EC: POST /api/events/{eventId}/checkin-submissions (multipart/form-data, field=file, contentType=image/jpeg)
    EC->>ES: submitCheckin(eventId, userId, multipartFile)

    ES->>EPR: findByEventIdAndUserId(eventId, userId)
    EPR->>DB: SELECT
    DB-->>EPR: participant

    alt participant == null
        ES-->>EC: BadRequestException("Синх viên chưa đăng ký sự kiện này")
        EC-->>F: 400
    else participant.checkedIn == true
        ES-->>EC: BadRequestException("Đã điểm danh rồi")
        EC-->>F: 400
    else có submission PENDING
        ES-->>EC: BadRequestException("Đang chờ duyệt")
        EC-->>F: 400
    else OK
        ES->>FS: validateAndStoreFile(file)
        FS->>FS: check contentType starts with "image/"
        FS->>FS: check extension jpg/jpeg/png
        FS->>FS: check size <= 5MB
        FS->>Disk: write file to upload-dir/event-checkins/
        Disk-->>FS: image_path
        FS-->>ES: image_path

        ES->>ECSR: save(new Submission{PENDING, submitted_at=now})
        ECSR->>DB: INSERT INTO event_checkin_submissions

        ES-->>EC: SubmissionResponse{id, status=PENDING}
        EC-->>F: 200 OK
        F->>M: Hiển thị "Chờ ban cán sự xác nhận"
    end
```

## 5.10. Camera Check-in: Admin duyệt / từ chối ảnh

```mermaid
sequenceDiagram
    actor A as Admin
    participant F as Flutter
    participant EC as EventCheckinSubmissionController
    participant ES as EventCheckinSubmissionService
    participant AS as AuthorizationService
    participant ECSR as EventCheckinSubmissionRepo
    participant EPR as EventParticipantRepo
    participant UR as UserRepo
    participant DB as MySQL

    A->>F: Mở EventParticipantsScreen → thấy "Chờ duyệt ảnh"
    A->>F: Xem ảnh (load từ /uploads/...)

    alt Admin bấm "Duyệt"
        F->>F: Confirm dialog
        A->>F: Xác nhận
        F->>EC: PUT /api/events/checkin-submissions/{submissionId}/approve
        EC->>ES: approveSubmission(submissionId, adminId)

        ES->>ECSR: findById(submissionId)
        ECSR-->>ES: submission

        ES->>AS: requireAdmin(adminId, event.classroomId)
        AS-->>ES: OK

        ES->>ECSR: save(submission.status=APPROVED, reviewedBy=admin, reviewedAt=now)
        ES->>EPR: save(participant.checkedIn=true, checkedInAt=now, checkedBy=admin)
        EPR->>DB: UPDATE event_participants

        ES-->>EC: SubmissionResponse{status=APPROVED}
        EC-->>F: 200 OK
        F->>A: Reload → Member chuyển "Đã điểm danh"
    else Admin bấm "Từ chối"
        F->>F: Dialog nhập lý do
        A->>F: Nhập lý do + xác nhận
        F->>EC: PUT /api/events/checkin-submissions/{submissionId}/reject {reason}
        EC->>ES: rejectSubmission(submissionId, adminId, reason)
        ES->>ECSR: save(status=REJECTED, rejectedReason=reason, reviewedBy=admin)
        ES-->>EC: SubmissionResponse{status=REJECTED}
        EC-->>F: 200 OK
        F->>A: Reload → Member thấy "Ảnh bị từ chối, gửi lại"
    end
```

## 5.11. Tổng kết

| Sequence | Tính nghiệp vụ chính được thể hiện |
|---|---|
| 5.1 Đăng nhập | Password hash + JWT generation |
| 5.2 Tạo khoản thu | Auto-sinh payment cho all members; requireAdmin |
| 5.3 Xác nhận thanh toán | Idempotency check; lưu confirmedBy |
| 5.4 QR + polling | Owner-only check; 5s polling; dispose timer |
| 5.5 Volunteer | requireMember; chống đăng ký trùng; lưu `source=VOLUNTEER` |
| 5.6 Assign participant | Admin xem tiến độ tối thiểu; validate member thuộc lớp; skip participant đã tồn tại; lưu `source=ASSIGNED` + audit |
| 5.7 Huỷ tham gia | Chặn `ASSIGNED`; `VOLUNTEER` chỉ huỷ được khi chưa check-in; dữ liệu cũ NULL fallback `VOLUNTEER` |
| 5.8 Check-in thủ công | requireAdmin; idempotency; lưu checkedBy |
| 5.9 Camera Check-in (gửi ảnh) | Multipart upload; validate contentType; FileStorageService; chống submit trùng |
| 5.10 Duyệt/Từ chối ảnh | requireAdmin; approve → set checkedIn; reject → lưu lý do; Member gửi lại được |

**Điểm chung của mọi sequence:**
- Mọi request (trừ /auth) đều đi qua `JwtAuthenticationFilter`.
- Service không tin user thông tin từ client — luôn check `AuthorizationService.requireMember/requireAdmin` trước action.
- Mọi action thay đổi state đều `@Transactional` để rollback nếu fail giữa chừng.
