---
layout: default
title: Account Verification & Activation Flow
permalink: /flows/account-verification-activation/
---

# Account Verification & Activation Flow

## 📋 Mục lục (Table of Contents)

### Phần chính
1. [**Tổng quan Flow**](#tổng-quan-flow) - Giới thiệu và 5 bước chính
2. [**Chi tiết nghiệp vụ từng bước**](./business-logic) - Phân tích sâu business logic
3. [**Technical Implementation - Backend**](./backend-implementation) - Code, API, Database
4. [**Technical Implementation - Frontend**](./frontend-implementation) - UI/UX, Forms, Upload
5. [**Security & Validation**](./security-validation) - Bảo mật và kiểm tra dữ liệu
6. [**Câu hỏi phản biện & Trả lời**](./defense-questions) - Chuẩn bị bảo vệ đồ án

---

## Tổng quan Flow

### Mục đích của Flow này

Flow **Account Verification & Activation** giải quyết bài toán quan trọng nhất của hệ thống: **Làm thế nào để xác minh danh tính người dùng trước khi cho phép họ sử dụng dịch vụ chia sẻ xe máy?**

**Vấn đề cần giải quyết:**
- Hệ thống chỉ phục vụ sinh viên đại học và tài xế có giấy phép hợp lệ
- Cần đảm bảo người dùng là ai họ nói (Identity Verification)
- Phải ngăn chặn tài khoản giả mạo, spam, hoặc lừa đảo
- Bảo vệ an toàn cho cả người đi và người chở

**Giải pháp:**
- Yêu cầu người dùng upload giấy tờ định danh (thẻ sinh viên, CMND, giấy phép lái xe)
- Admin thủ công xác minh từng hồ sơ
- Chỉ kích hoạt tài khoản sau khi hồ sơ được duyệt
- Gửi email thông báo cho người dùng

---

## 5 Bước chính của Flow

### Bước 1: User clicks "Sign Up" (Đăng ký tài khoản)

**Diễn ra điều gì:**
- User truy cập trang đăng ký
- Điền form: Email, Password, Full Name, Phone, Date of Birth
- Click nút "Sign Up"

**Kết quả:**
- Tạo record mới trong bảng `users`
- Status = `EMAIL_VERIFYING` (chờ xác thực email)
- User nhận email có mã OTP để xác thực

**Vai trò:**
- **User**: Người đăng ký (sinh viên hoặc tài xế)
- **System**: Backend API xử lý đăng ký

---

### Bước 2: User fills identity information with 2 pictures of identity card

**Diễn ra điều gì:**
- Sau khi xác thực email, user được yêu cầu upload giấy tờ
- Upload 2 ảnh thẻ sinh viên (mặt trước + mặt sau) hoặc CMND
- Điền thông tin: Student ID, University, Major (nếu là sinh viên)

**Kết quả:**
- File ảnh được upload lên storage (AWS S3 hoặc local)
- Tạo record trong bảng `verifications`:
  - `type` = `STUDENT_ID` (hoặc `DRIVER_LICENSE`)
  - `status` = `PENDING`
  - `document_url` = URL ảnh đã upload
- User status = `PENDING` (chờ admin duyệt)

**Vai trò:**
- **User**: Upload giấy tờ
- **Frontend**: Form upload file với preview
- **Backend**: API nhận file, validate, lưu vào storage

---

### Bước 3: Admin sees unverified account, manually checks information, and approves/declines

**Diễn ra điều gì:**
- Admin login vào Admin Portal
- Xem danh sách verification requests (status = PENDING)
- Click vào từng request để xem chi tiết:
  - Thông tin user: tên, email, phone
  - Ảnh thẻ sinh viên/CMND (phóng to để kiểm tra)
- So sánh thông tin trên ảnh với thông tin user đã điền
- Quyết định:
  - **Approve**: Nếu hợp lệ → chuyển sang Bước 4
  - **Reject**: Nếu không hợp lệ → gửi email từ chối + lý do

**Kết quả nếu Approve:**
- `verification.status` = `APPROVED`
- `verification.verified_by` = admin_id
- `verification.verified_at` = timestamp hiện tại
- `user.status` = `ACTIVE`
- Nếu là student: `rider_profile.status` = `ACTIVE`
- Nếu là driver: `driver_profile.status` = `INACTIVE` (chờ duyệt thêm giấy tờ khác)

**Kết quả nếu Reject:**
- `verification.status` = `REJECTED`
- `verification.rejection_reason` = lý do từ chối
- `user.status` vẫn là `PENDING`
- Gửi email thông báo từ chối cho user

**Vai trò:**
- **Admin**: Người kiểm tra và duyệt hồ sơ
- **Backend**: API approve/reject verification
- **Frontend Admin Portal**: UI để admin xem và duyệt

---

### Bước 4: Account verified and activated, system sends notification to user by registered email

**Diễn ra điều gì:**
- Sau khi admin approve, system tự động:
  - Cập nhật status trong database
  - Tạo wallet cho user (nếu là rider)
  - Gửi email thông báo "Tài khoản đã được kích hoạt"

**Nội dung email:**
- Tiêu đề: "Chúc mừng! Tài khoản của bạn đã được kích hoạt"
- Nội dung:
  - Xác nhận đã duyệt hồ sơ
  - Hướng dẫn đăng nhập
  - Link đến app/website
  - Thông tin liên hệ support

**Kết quả:**
- User nhận được email
- User biết tài khoản đã active
- User có thể đăng nhập và sử dụng dịch vụ

**Vai trò:**
- **Backend Email Service**: Gửi email tự động
- **Template Engine**: Render email HTML đẹp

---

### Bước 5: User signs in with activated account and uses service

**Diễn ra điều gì:**
- User mở app/website
- Đăng nhập bằng email + password
- System kiểm tra:
  - User tồn tại?
  - Password đúng?
  - Account status = `ACTIVE`?
- Nếu OK → tạo JWT token và trả về cho user

**Kết quả:**
- User nhận được JWT token
- Frontend lưu token vào localStorage
- User có thể:
  - **Nếu là Rider**: Đặt chuyến xe, xem lịch sử, nạp tiền
  - **Nếu là Driver**: Nhận yêu cầu chuyến xe, xem thu nhập

**Vai trò:**
- **User**: Đăng nhập
- **Backend**: Xác thực và cấp token
- **Frontend**: Lưu token và điều hướng đến Dashboard

---

## Why, What, How - Phân tích quyết định thiết kế

### WHY: Tại sao cần verification flow?

**Lý do nghiệp vụ:**
1. **An toàn**: Đảm bảo chỉ sinh viên thật và tài xế có giấy phép mới sử dụng
2. **Tin cậy**: Tạo niềm tin cho cộng đồng người dùng
3. **Pháp lý**: Tuân thủ quy định về bảo vệ dữ liệu cá nhân
4. **Chống gian lận**: Ngăn tài khoản ảo, spam, lừa đảo

**Lý do kỹ thuật:**
1. **Phân tách trách nhiệm**: User tự upload, Admin kiểm tra
2. **Audit trail**: Lưu lại ai duyệt, khi nào, lý do gì
3. **Scalability**: Có thể tích hợp AI/OCR sau này để tự động hóa
4. **Compliance**: Lưu giấy tờ để đối chiếu khi có tranh chấp

---

### WHAT: Flow này làm gì?

**Input:**
- Thông tin user: email, password, tên, phone
- Ảnh giấy tờ: thẻ sinh viên, CMND, giấy phép lái xe

**Process:**
- Validate dữ liệu
- Upload file lên storage
- Admin manual review
- Approve/Reject
- Send notification

**Output:**
- User account với status = ACTIVE
- Verification record với status = APPROVED
- Email notification
- User có thể login và dùng service

---

### HOW: Làm thế nào triển khai?

**Backend:**
- Spring Boot REST API
- JWT authentication
- File upload với MultipartFile
- Email service với Spring Mail + Thymeleaf
- PostgreSQL database
- Transaction management

**Frontend:**
- React TypeScript
- Form với React Hook Form
- File upload với preview
- Admin dashboard với table + modal
- API calls với Axios

**Infrastructure:**
- Docker containers
- AWS S3 để lưu ảnh
- RabbitMQ cho async email (optional)

---

## Điểm đặc biệt của Flow này

### 1. Manual Approval (Không tự động)

**Quyết định:** Admin phải thủ công duyệt từng hồ sơ

**Tại sao không tự động bằng AI/OCR?**
- **Chi phí**: API OCR (Google Vision, AWS Textract) tốn phí
- **Độ chính xác**: OCR có thể sai, đặc biệt với ảnh chất lượng kém
- **Bảo mật**: Thẻ sinh viên FPT có format đặc biệt, khó train model
- **Scale nhỏ**: Với vài trăm/nghìn user, manual review vẫn khả thi
- **Compliance**: Cần con người xác nhận để chịu trách nhiệm pháp lý

**Trade-off:**
- ✅ Độ chính xác cao
- ✅ Chi phí thấp
- ❌ Chậm (1-2 ngày)
- ❌ Không scale tốt khi có hàng triệu users

---

### 2. Two-step Registration

**Quyết định:** Tách đăng ký thành 2 bước (Create account → Upload documents)

**Tại sao không làm 1 bước?**
- **UX tốt hơn**: User không bị overwhelm với form dài
- **Tỷ lệ hoàn thành cao hơn**: Commit nhỏ dần thay vì bỏ cuộc giữa chừng
- **Xử lý lỗi dễ hơn**: Nếu upload fail, không mất thông tin đã điền
- **Email verification first**: Đảm bảo email thật trước khi upload giấy tờ

---

### 3. Multiple Verification Types

**Quyết định:** Hỗ trợ nhiều loại verification (STUDENT_ID, DRIVER_LICENSE, VEHICLE_REGISTRATION)

**Tại sao?**
- User có thể vừa là sinh viên, vừa là tài xế
- Mỗi role cần verify giấy tờ khác nhau
- Extensible: Dễ thêm loại verification mới sau này

**Database design:**
```
verifications table
- verification_id (PK)
- user_id (FK)
- type (enum: STUDENT_ID, DRIVER_LICENSE, VEHICLE_REGISTRATION)
- status (enum: PENDING, APPROVED, REJECTED)
- document_url (string)
```

---

## Sơ đồ tóm tắt (Simplified Flow Diagram)

```
[User] --> (1) Sign Up --> [Backend] --> Create User (status=EMAIL_VERIFYING)
                                          --> Send OTP Email
                                          
[User] --> (2) Verify Email --> [Backend] --> Update email_verified=true
                                           --> Status=PENDING
                                           
[User] --> (3) Upload Documents --> [Backend] --> Save to S3
                                              --> Create Verification (status=PENDING)
                                              
[Admin] --> (4) Review Request --> [Backend] --> Check documents
                                             --> Approve or Reject
                                             
[Backend] --> (5) If Approved --> Update status=ACTIVE
                              --> Create Wallet (if rider)
                              --> Send Email Notification
                              
[User] --> (6) Login --> [Backend] --> Check status=ACTIVE
                                   --> Generate JWT
                                   --> Return token
                                   
[User] --> (7) Use Service (Book ride, etc.)
```

---

## Next Steps - Đọc chi tiết

Bây giờ bạn đã hiểu tổng quan flow, hãy đọc từng phần chi tiết:

### 📘 [Chi tiết nghiệp vụ từng bước →](./business-logic)
Phân tích sâu business logic, validation rules, error handling cho từng bước

### 🔧 [Backend Implementation →](./backend-implementation)
Code examples, API endpoints, database schema, service layer, repository

### 🎨 [Frontend Implementation →](./frontend-implementation)
UI components, forms, file upload, admin dashboard, API integration

### 🔒 [Security & Validation →](./security-validation)
JWT authentication, file validation, XSS prevention, SQL injection, CSRF

### ❓ [Câu hỏi phản biện →](./defense-questions)
50+ câu hỏi thường gặp khi bảo vệ đồ án và cách trả lời chuyên nghiệp

---

[← Quay lại trang chủ]({{ site.baseurl }}/) | [Đọc tiếp: Business Logic →](./business-logic)
