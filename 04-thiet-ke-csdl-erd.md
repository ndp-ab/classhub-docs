# 04 — Thiết kế CSDL / ERD

## 4.1. Tổng quan

ClassHub sử dụng **MySQL 8** với 8 bảng. Mapping qua **JPA/Hibernate** (Spring Data JPA). Khi BE khởi động, `spring.jpa.hibernate.ddl-auto=update` tự sinh/cập nhật schema từ entity Java.

Database: `classhub_db` (charset `utf8mb4`).

## 4.2. Danh sách bảng

| # | Tên bảng | Mục đích | Số cột |
|---|---|---|---|
| 1 | `users` | Tài khoản người dùng | 6 |
| 2 | `classrooms` | Lớp học | 7 |
| 3 | `class_members` | Quan hệ user ↔ lớp + role | 5 |
| 4 | `fund_collections` | Đợt thu quỹ | 7 |
| 5 | `fund_payments` | Trạng thái đóng tiền của từng SV trong đợt thu | 8 |
| 6 | `fund_expenses` | Khoản chi | 7 |
| 7 | `events` | Sự kiện lớp | 8 |
| 8 | `event_participants` | Sinh viên đăng ký + check-in sự kiện | 7 |

## 4.3. Sơ đồ ERD (Mermaid)

```mermaid
erDiagram
    USERS ||--o{ CLASS_MEMBERS : "tham gia"
    USERS ||--o{ FUND_COLLECTIONS : "tạo (admin)"
    USERS ||--o{ FUND_PAYMENTS : "đóng tiền"
    USERS ||--o{ FUND_PAYMENTS : "xác nhận (admin)"
    USERS ||--o{ FUND_EXPENSES : "tạo (admin)"
    USERS ||--o{ EVENTS : "tạo (admin)"
    USERS ||--o{ EVENT_PARTICIPANTS : "đăng ký"
    USERS ||--o{ EVENT_PARTICIPANTS : "check-in (admin)"

    CLASSROOMS ||--o{ CLASS_MEMBERS : "có"
    CLASSROOMS ||--o{ FUND_COLLECTIONS : "có"
    CLASSROOMS ||--o{ FUND_EXPENSES : "có"
    CLASSROOMS ||--o{ EVENTS : "có"

    FUND_COLLECTIONS ||--o{ FUND_PAYMENTS : "sinh ra"
    EVENTS ||--o{ EVENT_PARTICIPANTS : "có"

    USERS {
        bigint id PK
        varchar full_name
        varchar email UK
        varchar password
        varchar avatar_url
        datetime created_at
    }

    CLASSROOMS {
        bigint id PK
        varchar class_name
        varchar faculty
        varchar academic_year
        varchar(8) invite_code UK
        bigint created_by FK
        datetime created_at
    }

    CLASS_MEMBERS {
        bigint id PK
        bigint user_id FK
        bigint classroom_id FK
        enum role "ADMIN|MEMBER"
        datetime joined_at
    }

    FUND_COLLECTIONS {
        bigint id PK
        varchar title
        decimal amount
        date deadline
        bigint classroom_id FK
        bigint created_by FK
        datetime created_at
    }

    FUND_PAYMENTS {
        bigint id PK
        bigint user_id FK
        bigint collection_id FK
        boolean is_paid
        boolean confirmed_by_admin
        datetime paid_at
        bigint confirmed_by FK
        varchar payment_code
    }

    FUND_EXPENSES {
        bigint id PK
        varchar title
        decimal amount
        varchar reason
        bigint classroom_id FK
        bigint created_by FK
        datetime created_at
    }

    EVENTS {
        bigint id PK
        varchar title
        text description
        varchar location
        datetime event_time
        bigint classroom_id FK
        bigint created_by FK
        datetime created_at
    }

    EVENT_PARTICIPANTS {
        bigint id PK
        bigint event_id FK
        bigint user_id FK
        boolean checked_in
        datetime checked_in_at
        bigint checked_by FK
        datetime registered_at
    }
```

## 4.4. Mô tả chi tiết từng bảng

### 4.4.1. `users` — Tài khoản người dùng

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | Khoá chính |
| `full_name` | VARCHAR(255) | NOT NULL | Họ tên đầy đủ |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE | Email đăng nhập |
| `password` | VARCHAR(255) | NOT NULL | Hash BCrypt (~60 ký tự) |
| `avatar_url` | VARCHAR(255) | NULL | URL ảnh đại diện (chưa dùng) |
| `created_at` | DATETIME | NOT NULL, không update | Thời điểm đăng ký |

**Lý do thiết kế:**
- `email UNIQUE` → đảm bảo không trùng tài khoản.
- Password lưu **hash BCrypt**, không lưu plaintext. Cụ thể BCrypt nhúng salt vào hash → không cần cột salt riêng.
- **Không** có cột `role` ở User → role được tách ra `class_members` để hỗ trợ user là Admin lớp này nhưng Member lớp khác.

### 4.4.2. `classrooms` — Lớp học

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | |
| `class_name` | VARCHAR(255) | NOT NULL | Vd "64KTPM3" |
| `faculty` | VARCHAR(255) | NULL | Khoa, vd "Công nghệ thông tin" |
| `academic_year` | VARCHAR(255) | NULL | Khoá, vd "K64" |
| `invite_code` | VARCHAR(8) | NOT NULL, UNIQUE | Mã mời 6 ký tự uppercase |
| `created_by` | BIGINT | NOT NULL | ID người tạo (legacy, dùng Long thay vì FK) |
| `created_at` | DATETIME | NOT NULL | |

**Lý do thiết kế:**
- `invite_code UNIQUE` để join lớp không nhầm.
- Code length 8 dù sinh ra 6 ký tự → phòng thay đổi format sau này.
- `created_by` hiện lưu kiểu `Long`, chưa phải `@ManyToOne` (lệch chuẩn với các entity khác — đã ghi trong known issues, đưa vào hướng phát triển để refactor).

### 4.4.3. `class_members` — Quan hệ user ↔ lớp + role

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `user_id` | BIGINT | FK → `users.id`, NOT NULL | |
| `classroom_id` | BIGINT | FK → `classrooms.id`, NOT NULL | |
| `role` | ENUM | NOT NULL | `ADMIN` hoặc `MEMBER` |
| `joined_at` | DATETIME | NOT NULL | |

**Ràng buộc:**
```sql
UNIQUE KEY (user_id, classroom_id)
```
→ Một user không thể có 2 bản ghi trong cùng 1 lớp.

**Lý do thiết kế:**
- **Bảng liên kết n-n** giữa users và classrooms. Một user có thể tham gia nhiều lớp. Một lớp có nhiều thành viên.
- Role nằm ở **đây**, không phải ở users → user có thể là Admin lớp K64KTPM3 nhưng là Member lớp K64KHMT (thực tế nếu họ tham gia cả 2 lớp).
- Đây là design quan trọng để chống lỗi "phân quyền toàn cục" mà hội đồng dễ bắt.

### 4.4.4. `fund_collections` — Đợt thu

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `title` | VARCHAR(255) | NOT NULL | Vd "Quỹ lớp tháng 5" |
| `amount` | DECIMAL(19,2) | NOT NULL | Số tiền (BigDecimal) |
| `deadline` | DATE | NULL | Hạn đóng (chỉ ngày, không cần giờ) |
| `classroom_id` | BIGINT | FK, NOT NULL | |
| `created_by` | BIGINT | FK → users.id, NOT NULL | Admin tạo |
| `created_at` | DATETIME | NOT NULL | |

**Lý do:**
- Dùng `BigDecimal` (DECIMAL trong SQL) cho tiền — `double` có lỗi làm tròn (`0.1 + 0.2 != 0.3`).
- `deadline` dùng `DATE` (LocalDate) — không cần giờ phút, chỉ ngày đến hạn.

### 4.4.5. `fund_payments` — Trạng thái đóng của từng SV

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `user_id` | BIGINT | FK, NOT NULL | Sinh viên |
| `collection_id` | BIGINT | FK → fund_collections, NOT NULL | Đợt thu nào |
| `is_paid` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `confirmed_by_admin` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `paid_at` | DATETIME | NULL | Khi admin xác nhận |
| `confirmed_by` | BIGINT | FK → users.id, NULL | **Audit B3:** admin nào xác nhận |
| `payment_code` | VARCHAR(255) | NULL | Mã nội dung chuyển khoản duy nhất |

**Lý do:**
- Sinh tự động khi tạo đợt thu (cho all members) và khi member join lớp muộn (cho all collections hiện có).
- 2 boolean `is_paid` và `confirmed_by_admin` hiện luôn cùng giá trị (logic redundant — known issue, gộp thành enum status đưa vào hướng phát triển).
- `payment_code` sinh ra lần đầu khi user mở QR, lưu lại để admin đối soát với sao kê ngân hàng. Format: `QUY{collectionId}-SV{userId}-{epochMillis}`.
- `confirmed_by` cho phép truy vết "ai đã xác nhận" — yêu cầu audit trail.

### 4.4.6. `fund_expenses` — Khoản chi

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `title` | VARCHAR(255) | NULL | |
| `amount` | DECIMAL(19,2) | NULL | |
| `reason` | VARCHAR(255) | NULL | Lý do chi |
| `classroom_id` | BIGINT | FK, NOT NULL | |
| `created_by` | BIGINT | FK → users.id, NOT NULL | |
| `created_at` | DATETIME | NOT NULL | |

### 4.4.7. `events` — Sự kiện

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `title` | VARCHAR(255) | NOT NULL | |
| `description` | TEXT | NULL | |
| `location` | VARCHAR(255) | NULL | |
| `event_time` | DATETIME | NOT NULL | Thời gian bắt đầu |
| `classroom_id` | BIGINT | FK, NOT NULL | |
| `created_by` | BIGINT | FK → users.id, NOT NULL | |
| `created_at` | DATETIME | NOT NULL | |

**Note:** Chưa có `end_time` — đưa vào hướng phát triển.

### 4.4.8. `event_participants` — Đăng ký + check-in

| Cột | Kiểu | Ràng buộc | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `event_id` | BIGINT | FK → events.id, NOT NULL | |
| `user_id` | BIGINT | FK → users.id, NOT NULL | |
| `checked_in` | BOOLEAN | NOT NULL DEFAULT FALSE | |
| `checked_in_at` | DATETIME | NULL | Thời điểm check-in |
| `checked_by` | BIGINT | FK → users.id, NULL | **Audit B4:** admin nào check-in |
| `registered_at` | DATETIME | NOT NULL | Khi sinh viên bấm đăng ký |

**Ràng buộc:**
```sql
UNIQUE KEY (event_id, user_id)
```
→ Một sinh viên không đăng ký 2 lần cho cùng 1 sự kiện. DB chặn ngay cả khi service có lỗi race condition.

## 4.5. Quan hệ giữa các bảng (giải thích)

| Quan hệ | Cardinality | Bảng liên kết | Ý nghĩa nghiệp vụ |
|---|---|---|---|
| User ↔ Classroom | n-n | `class_members` | Một user có thể tham gia nhiều lớp; một lớp có nhiều thành viên |
| Classroom 1 — n FundCollection | 1-n | (FK trực tiếp) | Một lớp có nhiều đợt thu |
| FundCollection 1 — n FundPayment | 1-n | (FK trực tiếp) | Một đợt thu sinh ra N bản ghi nợ (N = số thành viên) |
| User 1 — n FundPayment | 1-n | (FK trực tiếp) | Một sinh viên có nhiều bản ghi đóng tiền (qua nhiều đợt) |
| User 1 — n FundPayment (confirmedBy) | 1-n | (FK trực tiếp) | Một admin có thể đã xác nhận nhiều payment |
| Classroom 1 — n FundExpense | 1-n | | |
| Classroom 1 — n Event | 1-n | | |
| Event 1 — n EventParticipant | 1-n | | Một sự kiện có nhiều người đăng ký |
| User 1 — n EventParticipant | 1-n | | Một sinh viên đăng ký nhiều sự kiện |
| User 1 — n EventParticipant (checkedBy) | 1-n | | Audit ai check-in |

## 4.6. Quyết định thiết kế quan trọng

| Vấn đề | Quyết định | Lý do |
|---|---|---|
| Lưu role ở đâu? | Ở `class_members`, không ở `users` | Hỗ trợ multi-class với role khác nhau |
| Lưu tiền kiểu gì? | `BigDecimal` (DECIMAL) | Chính xác tuyệt đối, không có lỗi làm tròn |
| Mã mời sinh ra sao? | 6 ký tự uppercase từ UUID | Đủ chống đoán, đủ ngắn để gõ tay |
| Auto-sinh payment khi nào? | (1) Khi tạo collection (sinh cho all members), (2) Khi member join lớp (sinh cho all existing collections) | Đảm bảo không bỏ sót member nào |
| Audit trail | Lưu `confirmed_by` + `paid_at` ở payment; `checked_by` + `checked_in_at` ở participant | Hội đồng hỏi "ai làm" trả lời được |
| Chống đăng ký trùng | `UNIQUE(event_id, user_id)` + check ở service | 2 lớp bảo vệ: app + DB |
| FetchType cho FK | `LAZY` ở mọi `@ManyToOne` | Tránh load entity không cần thiết, performance tốt hơn |
| Timezone | `Asia/Ho_Chi_Minh` trong JDBC URL | Đảm bảo timestamp đúng giờ Việt Nam |

## 4.7. Script tạo DB (rút gọn)

```sql
CREATE DATABASE classhub_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'classhub_user'@'localhost' IDENTIFIED BY 'ClassHub@2026';
GRANT ALL PRIVILEGES ON classhub_db.* TO 'classhub_user'@'localhost';
```

Schema còn lại do **Hibernate `ddl-auto=update`** tự sinh khi khởi động Spring Boot lần đầu (đọc các `@Entity` Java).

## 4.8. Tổng kết

- **8 bảng** đầy đủ cho 3 phân hệ.
- Quan hệ n-n giữa User và Classroom giải quyết qua bảng `class_members` — đây là điểm "đúng OOAD" nhất của thiết kế.
- Audit trail (`confirmed_by`, `checked_by`) đảm bảo truy vết được mọi hành động.
- Sử dụng các kiểu dữ liệu phù hợp: BigDecimal cho tiền, LocalDate cho hạn, LocalDateTime cho timestamp.
