---
layout: default
title: Business Logic - Account Verification Flow
permalink: /flows/account-verification-activation/business-logic/
---

# Chi tiết nghiệp vụ từng bước

[← Quay lại tổng quan](../)

---

## 📋 Nội dung phần này

Phần này phân tích **sâu** business logic của từng bước trong flow, bao gồm:
- Điều kiện tiên quyết (preconditions)
- Quy tắc validation
- Xử lý edge cases
- Error handling
- State transitions

---

## Bước 1: User Registration (Đăng ký tài khoản)

### 1.1. User Input - Thông tin cần nhập

| Field | Type | Required | Validation Rules | Example |
|-------|------|----------|------------------|---------|
| `email` | String | ✅ Yes | - Format email hợp lệ<br>- Unique trong hệ thống<br>- Domain `.edu` hoặc `.edu.vn` (optional) | `student@fpt.edu.vn` |
| `password` | String | ✅ Yes | - Minimum 8 ký tự<br>- Có chữ hoa, chữ thường, số, ký tự đặc biệt<br>- Không chứa email/username | `SecurePass123!` |
| `fullName` | String | ✅ Yes | - 3-100 ký tự<br>- Chỉ chữ cái, khoảng trắng, dấu gạch ngang<br>- Không chứa số, ký tự đặc biệt | `Nguyễn Văn An` |
| `phone` | String | ✅ Yes | - Format Vietnam: `(+84|0)[0-9]{9,10}`<br>- Unique trong hệ thống | `0901234567` |
| `dateOfBirth` | Date | ❌ No | - Tuổi >= 18<br>- Không quá 100 tuổi | `2000-01-15` |
| `gender` | Enum | ❌ No | - MALE, FEMALE, OTHER | `MALE` |

### 1.2. Business Rules

#### Rule 1: Email phải unique
**Lý do:** Tránh duplicate accounts, email là identifier chính để login

**Validation:**
```java
if (userRepository.existsByEmail(email)) {
    throw ConflictException.emailAlreadyExists(email);
}
```

**Error message:** `"Email đã được sử dụng. Vui lòng dùng email khác hoặc đăng nhập."`

---

#### Rule 2: Phone phải unique
**Lý do:** Mỗi user chỉ có 1 số phone, dùng để xác thực OTP và liên lạc

**Validation:**
```java
String normalizedPhone = PhoneUtil.normalize(phone); // +84901234567
if (userRepository.existsByPhone(normalizedPhone)) {
    throw ConflictException.phoneAlreadyExists(phone);
}
```

**Edge case:** User nhập `0901234567` và `+84901234567` → coi là trùng

---

#### Rule 3: Password phải đủ mạnh
**Lý do:** Bảo vệ tài khoản khỏi bị brute-force

**Validation:**
```java
Pattern pattern = Pattern.compile(
    "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$"
);
if (!pattern.matcher(password).matches()) {
    throw ValidationException.weakPassword();
}
```

**Reject examples:**
- `password` → quá yếu
- `Password` → thiếu số và ký tự đặc biệt
- `Pass1!` → quá ngắn (<8)

---

#### Rule 4: Full Name không chứa ký tự nguy hiểm
**Lý do:** Prevent XSS attacks khi hiển thị name trên UI

**Validation:**
```java
String sanitized = HtmlUtils.htmlEscape(fullName);
if (!fullName.equals(sanitized)) {
    throw ValidationException.invalidFullName();
}
```

**Reject examples:**
- `<script>alert('xss')</script>` → XSS attack
- `John'; DROP TABLE users;--` → SQL injection attempt

---

### 1.3. Backend Processing Flow

```
1. Validate input (email format, password strength, etc.)
   ↓
2. Check email unique
   ↓
3. Check phone unique  
   ↓
4. Sanitize full name (prevent XSS)
   ↓
5. Hash password (BCrypt với salt)
   ↓
6. Create User entity
   - status = EMAIL_VERIFYING
   - email_verified = false
   - phone_verified = false
   - user_type = USER
   ↓
7. Save to database
   ↓
8. Generate OTP (6 digits, expire 10 minutes)
   ↓
9. Save OTP to database (table: otps)
   ↓
10. Send email với OTP
   ↓
11. Return RegisterResponse (user_id, email, token)
```

### 1.4. Database Changes

**Table: `users`**
```sql
INSERT INTO users (
    email, 
    phone, 
    password_hash, 
    full_name, 
    user_type, 
    status, 
    email_verified,
    phone_verified,
    created_at
) VALUES (
    'student@fpt.edu.vn',
    '+84901234567',
    '$2a$10$...', -- BCrypt hash
    'Nguyễn Văn An',
    'USER',
    'EMAIL_VERIFYING',
    false,
    false,
    NOW()
);
```

**Table: `otps`**
```sql
INSERT INTO otps (
    user_id,
    otp_code,
    otp_for,
    expires_at,
    created_at
) VALUES (
    123, -- user_id vừa tạo
    '784523',
    'EMAIL_VERIFICATION',
    NOW() + INTERVAL '10 minutes',
    NOW()
);
```

---

### 1.5. Email Notification

**Subject:** `Xác thực email đăng ký tài khoản MSSUS`

**Body:**
```
Xin chào Nguyễn Văn An,

Mã OTP của bạn là: 784523

Mã này có hiệu lực trong 10 phút.

Nếu bạn không yêu cầu đăng ký, vui lòng bỏ qua email này.

Trân trọng,
MSSUS Team
```

---

### 1.6. Error Scenarios

| Error | HTTP Status | Message | Solution |
|-------|-------------|---------|----------|
| Email đã tồn tại | 409 Conflict | `Email already exists` | Dùng email khác hoặc đăng nhập |
| Phone đã tồn tại | 409 Conflict | `Phone already exists` | Dùng số khác hoặc liên hệ support |
| Password yếu | 400 Bad Request | `Password too weak` | Nhập password theo quy tắc |
| Email format sai | 400 Bad Request | `Invalid email format` | Kiểm tra lại email |
| Full name có ký tự đặc biệt | 400 Bad Request | `Invalid full name` | Chỉ dùng chữ cái |
| User < 18 tuổi | 400 Bad Request | `Must be 18 or older` | Hệ thống chỉ dành cho 18+ |

---

### 1.7. State Diagram - User Status

```
[Not Exist] 
    ↓ (Sign Up)
[EMAIL_VERIFYING] 
    ↓ (Verify OTP)
[PENDING] 
    ↓ (Upload documents)
[PENDING] 
    ↓ (Admin approve)
[ACTIVE]
    ↓ (Violation/Ban)
[SUSPENDED]
```

---

## Bước 2: Email Verification

### 2.1. User Input

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `otp_code` | String | ✅ Yes | 6 digits | 
| `user_id` hoặc `email` | Integer/String | ✅ Yes | Valid user |

### 2.2. Business Rules

#### Rule 1: OTP phải hợp lệ và chưa expire
```java
Otp otp = otpRepository.findByUserIdAndOtpFor(userId, OtpFor.EMAIL_VERIFICATION)
    .orElseThrow(() -> new NotFoundException("OTP not found"));

if (otp.getExpiresAt().isBefore(LocalDateTime.now())) {
    throw new ValidationException("OTP expired");
}

if (!otp.getOtpCode().equals(inputOtp)) {
    throw new ValidationException("Invalid OTP");
}
```

#### Rule 2: OTP chỉ dùng được 1 lần
```java
if (otp.isUsed()) {
    throw new ValidationException("OTP already used");
}
```

#### Rule 3: Giới hạn số lần thử OTP sai
- Maximum 5 lần thử sai
- Sau 5 lần → block 30 phút

```java
int failedAttempts = otpRepository.countFailedAttempts(userId);
if (failedAttempts >= 5) {
    throw new RateLimitException("Too many failed attempts. Try again in 30 minutes");
}
```

### 2.3. Backend Processing

```
1. Validate OTP format (6 digits)
   ↓
2. Find OTP record in database
   ↓
3. Check OTP chưa expire
   ↓
4. Check OTP chưa used
   ↓
5. Check failed attempts < 5
   ↓
6. Compare OTP code
   ↓ (If match)
7. Update user.email_verified = true
   ↓
8. Update user.status = PENDING
   ↓
9. Mark OTP as used
   ↓
10. Return success message
```

### 2.4. Database Changes

```sql
-- Update user
UPDATE users 
SET email_verified = true,
    status = 'PENDING'
WHERE user_id = 123;

-- Mark OTP as used
UPDATE otps
SET used = true,
    used_at = NOW()
WHERE otp_id = 456;
```

---

## Bước 3: Upload Identity Documents

### 3.1. User Input

**Nếu là Student:**

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `document` | File[] | ✅ Yes | - Mảng 2 files (front + back)<br>- Format: JPG, PNG, PDF<br>- Size: Max 10MB per file<br>- Resolution: Min 800x600 |
| `student_id` | String | ❌ No | Format: SE######, SS######, etc. |
| `university` | String | ❌ No | Tên trường |
| `major` | String | ❌ No | Ngành học |

**Nếu là Driver:**

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `driver_license` | File[] | ✅ Yes | 2 files (front + back) |
| `license_number` | String | ✅ Yes | Số GPLX |
| `issue_date` | Date | ✅ Yes | Ngày cấp |
| `expiry_date` | Date | ✅ Yes | Ngày hết hạn (phải > today) |
| `license_class` | Enum | ✅ Yes | A1, A2, B1, B2, etc. |

### 3.2. File Validation Rules

#### Rule 1: File type phải hợp lệ
```java
List<String> allowedTypes = Arrays.asList("image/jpeg", "image/png", "application/pdf");
String contentType = file.getContentType();

if (!allowedTypes.contains(contentType)) {
    throw new ValidationException("File type not allowed. Only JPG, PNG, PDF accepted");
}
```

#### Rule 2: File size không quá 10MB
```java
long maxSize = 10 * 1024 * 1024; // 10MB
if (file.getSize() > maxSize) {
    throw new ValidationException("File too large. Maximum 10MB");
}
```

#### Rule 3: File không được rỗng
```java
if (file.isEmpty()) {
    throw new ValidationException("File is empty");
}
```

#### Rule 4: Ảnh phải có resolution tối thiểu
```java
BufferedImage image = ImageIO.read(file.getInputStream());
int width = image.getWidth();
int height = image.getHeight();

if (width < 800 || height < 600) {
    throw new ValidationException("Image resolution too low. Minimum 800x600");
}
```

### 3.3. File Upload Flow

```
1. Validate file type, size, resolution
   ↓
2. Generate unique filename
   - Format: {user_id}_{type}_{timestamp}_{random}.{ext}
   - Example: 123_student_id_1701706800_abc123.jpg
   ↓
3. Upload to Storage (S3 or local)
   ↓
4. Get public URL
   ↓
5. Save Verification record to database
   - type = STUDENT_ID
   - status = PENDING
   - document_url = URL của ảnh
   - document_type = IMAGE
   ↓
6. Return VerificationResponse
```

### 3.4. Database Changes

```sql
-- Nếu chưa có rider_profile, tạo mới
INSERT INTO rider_profiles (user_id, status, created_at)
VALUES (123, 'PENDING', NOW());

-- Tạo verification record
INSERT INTO verifications (
    user_id,
    type,
    status,
    document_url,
    document_type,
    created_at
) VALUES (
    123,
    'STUDENT_ID',
    'PENDING',
    'https://s3.amazonaws.com/mssus/uploads/123_student_id_front.jpg,https://s3.amazonaws.com/mssus/uploads/123_student_id_back.jpg',
    'IMAGE',
    NOW()
);
```

**Note:** `document_url` lưu nhiều URL, cách nhau bởi dấu phẩy

---

### 3.5. Edge Cases & Error Handling

#### Case 1: User upload ảnh mờ, không rõ
**Giải pháp:** 
- Frontend show preview để user tự kiểm tra
- Backend không validate độ nét (vì phức tạp)
- Admin sẽ reject và yêu cầu upload lại

#### Case 2: User upload ảnh không phải giấy tờ (meme, ảnh selfie, etc.)
**Giải pháp:**
- Backend không validate content (vì cần AI)
- Admin manual review sẽ phát hiện và reject

#### Case 3: User đã submit verification và đang pending
**Giải pháp:**
- Check trước khi cho phép upload lại
```java
if (verificationRepository.findByUserIdAndTypeAndStatus(
    userId, VerificationType.STUDENT_ID, VerificationStatus.PENDING
).isPresent()) {
    throw new ConflictException("You already have a pending verification request");
}
```

#### Case 4: File upload bị fail giữa chừng (network issue)
**Giải pháp:**
- Frontend retry tự động (3 lần)
- Nếu vẫn fail → show error và yêu cầu thử lại
- Backend không lưu partial data

---

### 3.6. Security Concerns

#### Concern 1: Malicious file upload (virus, malware)
**Giải pháp:**
- Validate file extension vs MIME type (check magic bytes)
- Scan file với antivirus (ClamAV) trước khi lưu
- Store files isolated, không execute

#### Concern 2: Path traversal attack
**Giải pháp:**
- Không dùng original filename từ user
- Generate filename mới với UUID
- Store file trong whitelist directory

```java
// ❌ BAD - Vulnerable
String filename = file.getOriginalFilename(); // User control
File dest = new File("/uploads/" + filename); // Can be ../../etc/passwd

// ✅ GOOD - Safe
String safeFilename = UUID.randomUUID().toString() + ".jpg";
File dest = new File("/uploads/verifications/" + safeFilename);
```

---

## Bước 4: Admin Review & Approve/Reject

### 4.1. Admin Input

| Action | Input Required | Validation |
|--------|----------------|------------|
| View list | `page`, `size`, `status` filter | - page >= 0<br>- size: 10, 20, 50 |
| View detail | `verification_id` | - Must exist<br>- Must be PENDING |
| Approve | `verification_id`, `notes` (optional) | - Must be PENDING<br>- Admin must be authenticated |
| Reject | `verification_id`, `rejection_reason` | - Must be PENDING<br>- Reason required (min 10 chars) |

### 4.2. Admin Portal UI Features

**List View:**
- Table với columns: User Name, Email, Type, Submitted Date, Status
- Filter by: Status (All, Pending, Approved, Rejected), Type, Date range
- Pagination: 20 items per page
- Sort by: Newest first (default), Oldest first, Name A-Z

**Detail Modal:**
- User info: Name, Email, Phone, Date of Birth
- Documents: Preview images (zoom in/out, fullscreen)
- Comparison: Info trên ảnh vs info user nhập
- Action buttons: Approve, Reject, Close
- Notes field: Admin có thể ghi chú

### 4.3. Approval Business Rules

#### Rule 1: Chỉ admin mới approve được
```java
@PreAuthorize("hasRole('ADMIN')")
public MessageResponse approveVerification(...) {
    // Logic
}
```

#### Rule 2: Không approve verification đã approved
```java
if (verification.getStatus() != VerificationStatus.PENDING) {
    throw new IllegalStateException(
        "Cannot approve verification with status: " + verification.getStatus()
    );
}
```

#### Rule 3: Admin phải là user hợp lệ
```java
User admin = userRepository.findByEmail(adminEmail)
    .orElseThrow(() -> new NotFoundException("Admin not found"));

if (admin.getUserType() != UserType.ADMIN) {
    throw new ForbiddenException("Only admins can approve verifications");
}
```

### 4.4. Approval Processing Flow

```
1. Validate admin có quyền approve
   ↓
2. Tìm verification record (by ID)
   ↓
3. Check status = PENDING
   ↓
4. Update verification:
   - status = APPROVED
   - verified_by = admin_id
   - verified_at = now()
   - metadata = admin notes
   ↓
5. Update user profile:
   - If STUDENT_ID → rider_profile.status = ACTIVE
   - If DRIVER_LICENSE → driver_profile.status = INACTIVE (chờ thêm giấy tờ)
   ↓
6. Update user status:
   - user.status = ACTIVE (nếu là student)
   ↓
7. Create wallet (nếu chưa có):
   - INSERT INTO wallets (user_id, balance, currency)
   ↓
8. Send email notification
   ↓
9. Return success message
```

### 4.5. Rejection Processing Flow

```
1. Validate admin có quyền reject
   ↓
2. Validate rejection_reason không rỗng
   ↓
3. Tìm verification record
   ↓
4. Update verification:
   - status = REJECTED
   - rejection_reason = input
   - verified_by = admin_id
   - verified_at = now()
   ↓
5. Update profile status:
   - rider_profile.status = REJECTED hoặc
   - driver_profile.status = REJECTED
   ↓
6. Send email notification with reason
   ↓
7. Return success message
```

### 4.6. Database Changes (Approve)

```sql
-- Update verification
UPDATE verifications
SET status = 'APPROVED',
    verified_by = 999, -- admin user_id
    verified_at = NOW(),
    metadata = 'Giấy tờ hợp lệ, thông tin khớp'
WHERE verification_id = 123;

-- Update rider profile (nếu là student)
UPDATE rider_profiles
SET status = 'ACTIVE',
    activated_at = NOW()
WHERE user_id = 456;

-- Update user status
UPDATE users
SET status = 'ACTIVE'
WHERE user_id = 456;

-- Create wallet (nếu chưa có)
INSERT INTO wallets (user_id, balance, currency, created_at)
VALUES (456, 0, 'VND', NOW())
ON CONFLICT (user_id) DO NOTHING;
```

### 4.7. Database Changes (Reject)

```sql
-- Update verification
UPDATE verifications
SET status = 'REJECTED',
    rejection_reason = 'Ảnh mờ, không rõ thông tin. Vui lòng chụp lại với ánh sáng tốt hơn',
    verified_by = 999,
    verified_at = NOW()
WHERE verification_id = 123;

-- Update rider profile
UPDATE rider_profiles
SET status = 'REJECTED'
WHERE user_id = 456;
```

### 4.8. Rejection Reasons Examples

| Reason Code | Vietnamese Message | English Message |
|-------------|-------------------|-----------------|
| `BLURRY_IMAGE` | Ảnh mờ, không rõ thông tin | Image is blurry, cannot read information |
| `INFO_MISMATCH` | Thông tin không khớp với giấy tờ | Information does not match document |
| `INVALID_DOC` | Giấy tờ không hợp lệ hoặc đã hết hạn | Document invalid or expired |
| `FAKE_DOC` | Nghi ngờ giấy tờ giả mạo | Suspected fake document |
| `WRONG_TYPE` | File upload không phải giấy tờ cần thiết | Wrong document type uploaded |
| `INCOMPLETE` | Thiếu mặt trước hoặc mặt sau | Missing front or back side |

---

## Bước 5: Email Notification & User Login

### 5.1. Email Template (Approved)

**Subject:** `✅ Tài khoản MSSUS của bạn đã được kích hoạt!`

**HTML Body:**
```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); 
                  color: white; padding: 30px; text-align: center; }
        .content { background: white; padding: 30px; }
        .button { background: #10b981; color: white; padding: 12px 30px;
                  text-decoration: none; border-radius: 5px; display: inline-block; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🎉 Chúc mừng!</h1>
            <p>Tài khoản của bạn đã được kích hoạt</p>
        </div>
        <div class="content">
            <p>Xin chào <strong>{{ fullName }}</strong>,</p>
            
            <p>Chúng tôi vui mừng thông báo rằng hồ sơ xác minh của bạn đã được phê duyệt thành công!</p>
            
            <h3>Thông tin tài khoản:</h3>
            <ul>
                <li>Email: {{ email }}</li>
                <li>Loại tài khoản: {{ userType }}</li>
                <li>Ngày kích hoạt: {{ approvalDate }}</li>
            </ul>
            
            <h3>Bước tiếp theo:</h3>
            <ol>
                <li>Đăng nhập vào hệ thống</li>
                <li>Hoàn thiện hồ sơ cá nhân</li>
                <li>Bắt đầu sử dụng dịch vụ</li>
            </ol>
            
            <p style="text-align: center; margin: 30px 0;">
                <a href="{{ frontendUrl }}/login" class="button">
                    Đăng nhập ngay
                </a>
            </p>
            
            <p>Nếu bạn có thắc mắc, vui lòng liên hệ: <a href="mailto:support@mssus.com">support@mssus.com</a></p>
            
            <p>Trân trọng,<br>MSSUS Team</p>
        </div>
    </div>
</body>
</html>
```

### 5.2. Email Template (Rejected)

**Subject:** `❌ Yêu cầu xác minh của bạn cần được cập nhật`

**Body:**
```
Xin chào {{ fullName }},

Rất tiếc, hồ sơ xác minh của bạn chưa được phê duyệt vì lý do sau:

{{ rejectionReason }}

Bạn vui lòng:
1. Kiểm tra và chuẩn bị lại giấy tờ theo yêu cầu
2. Đăng nhập vào tài khoản
3. Upload lại giấy tờ hợp lệ

Lưu ý:
- Ảnh phải rõ nét, độ phân giải tối thiểu 800x600
- Chụp trong điều kiện ánh sáng tốt
- Đảm bảo thông tin trên ảnh khớp với thông tin đăng ký
- Upload cả 2 mặt của giấy tờ

Nếu cần hỗ trợ, liên hệ: support@mssus.com

Trân trọng,
MSSUS Team
```

### 5.3. User Login Flow

```
1. User nhập email + password
   ↓
2. Backend validate:
   - User exists?
   - Password correct?
   - Status = ACTIVE?
   ↓
3. If validation pass:
   - Generate JWT token
   - Payload: { user_id, email, roles, active_profile }
   - Expiry: 24 hours (access token), 30 days (refresh token)
   ↓
4. Return LoginResponse:
   - access_token
   - refresh_token
   - user info (id, name, email, roles)
   - expires_in: 86400 seconds
   ↓
5. Frontend save token to localStorage
   ↓
6. Redirect to Dashboard
```

### 5.4. Login Validation Rules

#### Rule 1: Account phải active
```java
if (user.getStatus() != UserStatus.ACTIVE) {
    String message = switch (user.getStatus()) {
        case EMAIL_VERIFYING -> "Please verify your email first";
        case PENDING -> "Your account is pending verification";
        case SUSPENDED -> "Your account has been suspended. Contact support";
        case BANNED -> "Your account has been banned";
        default -> "Account not active";
    };
    throw new UnauthorizedException(message);
}
```

#### Rule 2: Password attempts limited
- Maximum 5 lần thử sai liên tiếp
- Sau 5 lần → lock account 30 minutes

```java
int failedAttempts = loginAttemptService.getFailedAttempts(email);
if (failedAttempts >= 5) {
    throw new TooManyRequestsException("Too many failed login attempts. Please try again in 30 minutes");
}
```

#### Rule 3: JWT token phải valid
- Signature đúng
- Chưa expire
- User chưa bị ban/suspend
- Token version match (để revoke token khi change password)

### 5.5. Edge Cases

#### Case 1: User verify email nhưng chưa upload documents
**Behavior:**
- Cho phép login
- Sau khi login → redirect đến trang "Complete your profile"
- Show alert: "Please upload identity documents to use services"

#### Case 2: User upload documents nhưng bị reject
**Behavior:**
- Cho phép login
- Sau login → show notification: "Your verification was rejected. Please resubmit"
- Redirect đến upload page với rejection reason hiển thị

#### Case 3: User có multiple profiles (vừa rider, vừa driver)
**Behavior:**
- Sau login → show profile picker: "Continue as Rider" or "Continue as Driver"
- Generate token với role tương ứng
- User có thể switch profile sau

---

## Tổng kết Business Logic

### State Transitions Summary

```
User Status Flow:
NOT_EXIST → EMAIL_VERIFYING → PENDING → ACTIVE → (SUSPENDED/BANNED)

Verification Status Flow:
PENDING → APPROVED/REJECTED

Profile Status Flow (Rider):
PENDING → ACTIVE/REJECTED

Profile Status Flow (Driver):
PENDING → INACTIVE → ACTIVE/REJECTED
         ↑
         (Chờ thêm vehicle registration)
```

### Critical Business Rules Recap

1. **Email và Phone phải unique** - Tránh duplicate accounts
2. **Email verification trước document upload** - Đảm bảo email thật
3. **Admin manual approval bắt buộc** - Đảm bảo chất lượng verification
4. **File validation nghiêm ngặt** - Bảo mật và quality control
5. **Status transitions tuân thủ flow** - Không skip bước
6. **Notification cho mọi action quan trọng** - User luôn biết status
7. **Audit trail đầy đủ** - Lưu ai approve, khi nào, lý do gì

---

[← Quay lại tổng quan](../account-verification-activation/) | [Đọc tiếp: Backend Implementation →](../backend-implementation)
