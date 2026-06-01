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
| Thiết kế ERD (8 bảng) | ✅ |
| Xác định 3 actor (Guest/Member/Admin) | ✅ |
| Lập danh sách 21 endpoint dự kiến | ✅ |
| Use Case Diagram | ✅ (text-based + Mermaid) |
| Wireframe Figma | 🟡 Backfill (làm sau code) |

## Giai đoạn 2 — Backend core (Tuần 3-5)

| Việc | Trạng thái |
|---|---|
| Setup Spring Boot project + MySQL | ✅ |
| 8 entity + repository | ✅ |
| Auth: register/login + BCrypt + JWT generation | ✅ |
| Classroom: create/join/list | ✅ |
| Fund collections + payments + auto-sinh | ✅ |
| QR VietQR + polling status | ✅ |
| Fund expenses | ✅ |
| Event + participant + check-in | ✅ |
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
| Event tab + create + participants | ✅ |
| Models có defensive parsing | ✅ |

**Vấn đề gặp:**
- Android emulator dùng `10.0.2.2` thay vì `localhost` — phải hardcode `baseUrl` đổi tuỳ thiết bị.
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

## Giai đoạn 7 — Bổ sung trước demo (kế hoạch)

| Việc | Ưu tiên | Effort |
|---|---|---|
| API `/classrooms/{id}/members` | 🔴 Cao | 30 phút |
| API `/fund-statistics` | 🔴 Cao | 1 tiếng |
| FE tab Thành viên (sau khi BE có API) | 🟡 | 30 phút |
| FE Dashboard quỹ (widget tổng quan) | 🟡 | 1-2 tiếng |
| Smoke test BE bằng Postman | 🔴 | 15 phút |
| Test E2E 13 bước demo trên 2 thiết bị | 🔴 | 30 phút |
| Đổi `vietqr.account-no` thành TK thật | 🔴 | 1 phút |
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
Trước demo:      API members + statistics + smoke test
```

## Số liệu thống kê

| Hạng mục | Số |
|---|---|
| File source BE (Java) | 52 |
| File source FE (Dart) | ~22 |
| Endpoint REST | 21 |
| Bảng DB | 8 |
| Screen Flutter | 13 |
| File documentation | 14 |
| Test case đặc tả | 28 |
| Bug critical đã fix | 9 (B1–B8 + freeze CollectionPaymentsScreen) |
| Tổng thời gian phát triển | ~7 tuần + 2 ngày design system polish |

## Bài học rút ra

1. **Audit sớm hơn**: Nếu audit theo checklist từ tuần 4 thay vì tuần 6 thì không phải refactor nhiều ở B1+B2.
2. **Test sớm hơn**: Manual test với Postman từng phân hệ ngay sau khi viết — không phải để tới khi tích hợp FE mới phát hiện bug.
3. **Lưu thông tin về quyết định thiết kế**: Tại sao chọn Provider thay Bloc, tại sao không có OWNER role... Để trả lời hội đồng dễ.
4. **Hibernate `ddl-auto=update`** rất tiện nhưng phải backup DB trước khi đổi entity quan trọng.
5. **Confirm dialog** cho action destructive đáng làm — chỉ 10 dòng code nhưng chống lỗi UX rõ rệt.
