# ClassHub — Tài liệu đồ án

> **Tên đề tài:** Xây dựng hệ thống ClassHub hỗ trợ quản lý quỹ lớp, hồ sơ sinh viên và sự kiện nội bộ lớp hành chính
> **Sinh viên:** Nguyễn Duy Phong — K64KTPM3 — MSV 2251172450
> **GVHD:** Th.S. Tạ Chí Hiếu
> **Loại:** Đồ án tốt nghiệp (Solo project)

---

## Mục lục tài liệu

| # | Tài liệu | Nội dung |
|---|---|---|
| [01](01-khao-sat-bai-toan.md) | Khảo sát bài toán | Hiện trạng quản lý lớp bằng Zalo + Excel + Forms; vấn đề rút ra; nhu cầu Admin/Member |
| [02](02-yeu-cau-he-thong.md) | Yêu cầu hệ thống | 32 yêu cầu chức năng + 26 yêu cầu phi chức năng + giới hạn phạm vi |
| [03](03-actor-usecase.md) | Actor & Use Case | 3 actor (Guest/Member/Admin); 23 use case; sơ đồ Mermaid |
| [04](04-thiet-ke-csdl-erd.md) | Thiết kế CSDL / ERD | 8 bảng, quan hệ n-n qua ClassMember; sơ đồ ERD Mermaid |
| [05](05-sequence-diagram.md) | Sequence Diagram | 6 luồng quan trọng (login, tạo khoản thu, xác nhận, QR+polling, volunteer, check-in) |
| [06](06-kien-truc-he-thong.md) | Kiến trúc hệ thống | 3 lớp client-server; 5 tầng backend; tech stack; layered architecture |
| [07](07-api-documentation.md) | API Documentation | 21 endpoint REST chi tiết: request/response/status/quyền |
| [08](08-backend-documentation.md) | Backend Documentation | Cấu    trúc package; service/repository/entity/dto; convention |
| [09](09-frontend-documentation.md) | Frontend Documentation | 13 màn hình; Provider; service layer; navigation flow |
| [10](10-kiem-thu.md) | Kiểm thử | 28 test case + hướng dẫn chạy Postman/JUnit + bug history |
| [11](11-huong-dan-cai-dat.md) | Hướng dẫn cài đặt | Setup BE + DB + FE + troubleshoot |
| [12](12-demo-script.md) | Demo Script | 10 bước demo + câu hỏi hội đồng + backup plan |
| [13](13-tai-khoan-test.md) | Tài khoản test | Script setup nhanh trước demo |
| [14](14-progress-log.md) | Progress Log | Timeline phát triển + bài học rút ra |

---

## Cách đọc tài liệu

### 👨‍🏫 Nếu bạn là **giáo viên hướng dẫn / phản biện**
Đọc theo thứ tự:
1. **01** + **02**: Hiểu tại sao cần ClassHub.
2. **03** + **04**: Hiểu thiết kế hệ thống.
3. **06** + **07**: Hiểu kỹ thuật.
4. **10**: Hiểu mức độ kiểm thử.
5. **12**: Xem demo.

### 👨‍💻 Nếu bạn là **developer mới tiếp nhận project**
Đọc theo thứ tự:
1. **11**: Cài đặt môi trường, chạy được app.
2. **06**: Hiểu kiến trúc.
3. **08** + **09**: Hiểu code BE và FE.
4. **07**: API contract.
5. **14**: Lịch sử để hiểu context.

### 🎤 Nếu bạn cần **chuẩn bị bảo vệ**
Đọc theo thứ tự:
1. **12**: Demo script — học thuộc.
2. **03**: Use case + câu hỏi hội đồng có thể hỏi.
3. **10**: Test case + bug history (để chứng minh đã kiểm thử).
4. **14**: Trả lời "em làm trong bao lâu, gặp khó khăn gì".

---

## Quick links

- **Backend repo:** https://github.com/agio7/classhub-api
- **Frontend repo:** https://github.com/agio7/classhub-app
- **Backend audit:** [`classhub-api/documents/BACKEND_AUDIT.md`](../classhub-api/documents/BACKEND_AUDIT.md)
- **Backend fix log:** [`classhub-api/documents/BACKEND_FIX_LOG.md`](../classhub-api/documents/BACKEND_FIX_LOG.md)
- **Project status:** [`classhub-api/documents/PROJECT_STATUS.md`](../classhub-api/documents/PROJECT_STATUS.md)
- **Frontend progress:** [`classhub_app/documents/FRONTEND_FUND_PROGRESS.md`](../classhub_app/documents/FRONTEND_FUND_PROGRESS.md)

---

## Trạng thái dự án (cập nhật 2026-05-23)

### Đã hoàn thành
- ✅ 3 phân hệ chính: Auth, Quỹ lớp (gồm QR), Sự kiện
- ✅ 21/21 endpoint REST
- ✅ 13/13 màn Flutter
- ✅ Bảo mật + Audit trail (sau B1–B8)
- ✅ Manual test E2E qua demo script
- ✅ Design system FE (theme tokens + 9 shared widgets) + refactor 13 màn
- ✅ HomeScreen class selector + ClassroomDetailScreen workspace layout
- ✅ Classroom switcher bottom sheet trong workspace
- ✅ Fix bug freeze CollectionPaymentsScreen trên Chrome Web (Tier 4 refactor)

### Còn lại trước demo
- 🔴 Smoke test BE qua Postman (15 phút)
- 🔴 Đổi `vietqr.account-no` thành TK ngân hàng thật (1 phút)
- 🔴 Test trên thiết bị thật (15 phút)

### Sẽ làm nếu còn thời gian
- 🟡 API `/classrooms/{id}/members` + tab Thành viên FE (1h tổng)
- 🟡 API `/fund-statistics` + Dashboard FE (1-2h)
- 🟡 Test case JUnit + MockMvc (3-4h)
- 🟡 Postman collection export

### Hướng phát triển (đưa vào báo cáo)
- Role OWNER là hướng mở rộng sau này, chưa có trong code BE/FE hiện tại
- Push notification (FCM)
- Đối soát ngân hàng tự động (webhook VietQR)
- iOS support
- Refresh token

---

## Cấu trúc thư mục dự án

```
D:/big_dream/
├── classhub-api/                  # Spring Boot backend
│   ├── src/main/java/com/classhub/classhubapi/
│   ├── src/main/resources/application.properties
│   ├── documents/                 # BE-specific docs (audit, fix log, status)
│   └── pom.xml
│
├── classhub_app/                  # Flutter frontend
│   ├── lib/
│   ├── documents/                 # FE-specific docs (progress)
│   └── pubspec.yaml
│
└── docs/                          # 📍 Tài liệu chính (bạn đang ở đây)
    ├── README.md                  # File này
    ├── 01-khao-sat-bai-toan.md
    ├── 02-yeu-cau-he-thong.md
    ├── 03-actor-usecase.md
    ├── 04-thiet-ke-csdl-erd.md
    ├── 05-sequence-diagram.md
    ├── 06-kien-truc-he-thong.md
    ├── 07-api-documentation.md
    ├── 08-backend-documentation.md
    ├── 09-frontend-documentation.md
    ├── 10-kiem-thu.md
    ├── 11-huong-dan-cai-dat.md
    ├── 12-demo-script.md
    ├── 13-tai-khoan-test.md
    └── 14-progress-log.md
```

---

## Stack công nghệ

| Lớp | Công nghệ |
|---|---|
| Frontend | Flutter (Dart), Provider, http, shared_preferences, custom design system (Material 3 + tokens + shared widgets ở `lib/core/`) |
| Backend | Spring Boot 4.0, Spring Security, JPA/Hibernate, jjwt |
| Database | MySQL 8 |
| Build tool | Maven (mvnw) |
| External | VietQR (img.vietqr.io) |

---

## Tổng kết

ClassHub là sản phẩm phục vụ **bài toán thực tế** mà sinh viên đại học đang gặp phải. Hệ thống không chỉ là một tập hợp các chức năng, mà là quy trình **khảo sát → phân tích → thiết kế → cài đặt → kiểm thử → đánh giá** đầy đủ — đúng yêu cầu của một đồ án tốt nghiệp.

Đặc điểm nổi bật:
- **Bảo mật chặt**: JWT validate + authorization per-class + audit trail.
- **Dữ liệu chính xác**: BigDecimal cho tiền, unique constraint chống trùng, idempotency.
- **Trải nghiệm thực tế**: QR thanh toán + polling cảm giác real-time + confirm dialog.
- **Mở rộng được**: Layered architecture, tách DTO/Entity, role tách theo lớp.
