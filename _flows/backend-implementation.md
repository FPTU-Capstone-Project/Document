---
layout: default
title: Backend Implementation - Account Verification Flow
permalink: /flows/backend-implementation/
---

# Backend Implementation Chi Tiết

[← Quay lại tổng quan]({{ site.baseurl }}/flows/account-verification-activation/)

---

## 📋 Nội dung phần này

Phần này giải thích **chi tiết kỹ thuật** về cách triển khai backend, bao gồm:
- Kiến trúc tổng thể (Architecture)
- Database schema
- API endpoints với examples
- Service layer implementation
- Code examples thực tế từ project
- Security implementation
- Error handling patterns

---

## Architecture Overview

### Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Framework** | Spring Boot 3.2 | REST API framework |
| **Security** | Spring Security + JWT | Authentication & Authorization |
| **Database** | PostgreSQL 15 | Relational data storage |
| **ORM** | JPA + Hibernate | Object-relational mapping |
| **Validation** | Jakarta Validation | Input validation |
| **File Storage** | AWS S3 / Local | Document storage |
| **Email** | Spring Mail + Thymeleaf | Email templates |
| **Logging** | SLF4J + Logback | Application logs |

### Layered Architecture

```
┌─────────────────────────────────────┐
│     Controller Layer                │  ← REST endpoints
│  - AuthController                   │
│  - ProfileController                │
│  - VerificationController           │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│     Service Layer                   │  ← Business logic
│  - AuthService                      │
│  - ProfileService                   │
│  - VerificationService              │
│  - EmailService                     │
│  - FileUploadService                │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│     Repository Layer                │  ← Data access
│  - UserRepository                   │
│  - VerificationRepository           │
│  - RiderProfileRepository           │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│     Database (PostgreSQL)           │
└─────────────────────────────────────┘
```

---

## Database Schema

### Table: `users`

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(100) NOT NULL,
    student_id VARCHAR(50) UNIQUE,
    date_of_birth DATE,
    gender VARCHAR(20),
    user_type VARCHAR(20) NOT NULL DEFAULT 'USER',
    profile_photo_url TEXT,
    email_verified BOOLEAN DEFAULT FALSE,
    phone_verified BOOLEAN DEFAULT FALSE,
    token_version INTEGER DEFAULT 1,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (user_type IN ('USER', 'ADMIN')),
    CHECK (status IN ('EMAIL_VERIFYING', 'PENDING', 'ACTIVE', 'SUSPENDED', 'BANNED'))
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_status ON users(status);
```

**Giải thích columns:**
- `user_id`: Primary key, auto increment
- `email`: Unique, dùng để login
- `phone`: Unique, dùng OTP verification
- `password_hash`: BCrypt hashed password
- `token_version`: Để revoke tất cả tokens khi change password
- `status`: Track user lifecycle state
- `user_type`: USER (normal) hoặc ADMIN

---

### Table: `rider_profiles`

```sql
CREATE TABLE rider_profiles (
    rider_id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'PENDING',
    activated_at TIMESTAMP,
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_rides INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (status IN ('PENDING', 'ACTIVE', 'SUSPENDED', 'REJECTED'))
);

CREATE INDEX idx_rider_user_id ON rider_profiles(user_id);
```

**Giải thích:**
- Quan hệ `1:1` với `users` (one user → one rider_profile)
- `status`: Track verification status riêng cho rider role
- `rating`, `total_rides`: Metrics cho rider

---

### Table: `driver_profiles`

```sql
CREATE TABLE driver_profiles (
    driver_id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    status VARCHAR(20) DEFAULT 'PENDING',
    activated_at TIMESTAMP,
    license_number VARCHAR(50),
    license_class VARCHAR(10),
    license_issue_date DATE,
    license_expiry_date DATE,
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_rides INTEGER DEFAULT 0,
    earnings DECIMAL(12,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (status IN ('PENDING', 'INACTIVE', 'ACTIVE', 'SUSPENDED', 'REJECTED'))
);

CREATE INDEX idx_driver_user_id ON driver_profiles(user_id);
```

**Giải thích:**
- Quan hệ `1:1` với `users`
- `status`: INACTIVE = đã duyệt giấy phép nhưng chưa duyệt xe
- Lưu thông tin giấy phép lái xe

---

### Table: `verifications`

```sql
CREATE TABLE verifications (
    verification_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    document_url TEXT,
    document_type VARCHAR(20),
    rejection_reason TEXT,
    verified_by INTEGER REFERENCES users(user_id),
    verified_at TIMESTAMP,
    expires_at TIMESTAMP,
    metadata TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (type IN ('STUDENT_ID', 'DRIVER_LICENSE', 'DRIVER_DOCUMENTS', 'VEHICLE_REGISTRATION', 'BACKGROUND_CHECK')),
    CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED', 'EXPIRED')),
    CHECK (document_type IN ('IMAGE', 'PDF'))
);

CREATE INDEX idx_verifications_user_type_status ON verifications(user_id, type, status);
CREATE INDEX idx_verifications_status_created ON verifications(status, created_at);
```

**Giải thích:**
- Lưu tất cả verification requests
- `document_url`: URL ảnh đã upload (có thể nhiều URL, cách nhau bởi dấu phẩy)
- `verified_by`: Admin nào approve/reject
- `metadata`: JSON string chứa extra info (admin notes, etc.)

---

### Table: `wallets`

```sql
CREATE TABLE wallets (
    wallet_id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    balance DECIMAL(12,2) DEFAULT 0 NOT NULL,
    currency VARCHAR(3) DEFAULT 'VND',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (balance >= 0)
);

CREATE INDEX idx_wallets_user_id ON wallets(user_id);
```

---

### Table: `otps`

```sql
CREATE TABLE otps (
    otp_id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    otp_code VARCHAR(6) NOT NULL,
    otp_for VARCHAR(50) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    used BOOLEAN DEFAULT FALSE,
    used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    CHECK (otp_for IN ('EMAIL_VERIFICATION', 'PHONE_VERIFICATION', 'PASSWORD_RESET'))
);

CREATE INDEX idx_otps_user_otp_for ON otps(user_id, otp_for);
```

---

## Entity Classes

### User Entity

```java
@Entity
@Table(name = "users")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "user_id")
    private Integer userId;
    
    @Column(name = "email", unique = true, nullable = false)
    private String email;
    
    @Column(name = "phone", unique = true)
    private String phone;
    
    @Column(name = "password_hash")
    private String passwordHash;
    
    @Column(name = "full_name", nullable = false)
    private String fullName;
    
    @Column(name = "student_id", unique = true)
    private String studentId;
    
    @Column(name = "date_of_birth")
    private LocalDate dateOfBirth;
    
    private String gender;
    
    @Column(name = "user_type")
    @Enumerated(EnumType.STRING)
    @Builder.Default
    private UserType userType = UserType.USER;
    
    @Column(name = "profile_photo_url")
    private String profilePhotoUrl;
    
    @Column(name = "email_verified")
    @Builder.Default
    private Boolean emailVerified = false;
    
    @Column(name = "phone_verified")
    @Builder.Default
    private Boolean phoneVerified = false;
    
    @Column(name = "token_version")
    @Builder.Default
    private int tokenVersion = 1;
    
    @Column(name = "status")
    @Enumerated(EnumType.STRING)
    @Builder.Default
    private UserStatus status = UserStatus.PENDING;
    
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    // Relationships
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private RiderProfile riderProfile;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private DriverProfile driverProfile;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private Wallet wallet;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Verification> verifications;
}
```

**Design decisions:**
- `@Builder`: Sử dụng Builder pattern để tạo object dễ dàng
- `FetchType.LAZY`: Không load relationships khi không cần (performance)
- `cascade = CascadeType.ALL`: Khi delete user, tự động delete profiles và verifications
- `@CreatedDate`, `@LastModifiedDate`: JPA Auditing tự động set timestamps

---

### Verification Entity

```java
@Entity
@Table(name = "verifications")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Verification {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "verification_id")
    private Integer verificationId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(name = "type", nullable = false)
    @Enumerated(EnumType.STRING)
    private VerificationType type;
    
    @Column(name = "status")
    @Enumerated(EnumType.STRING)
    @Builder.Default
    private VerificationStatus status = VerificationStatus.PENDING;
    
    @Column(name = "document_url")
    private String documentUrl;
    
    @Column(name = "document_type")
    @Enumerated(EnumType.STRING)
    private DocumentType documentType;
    
    @Column(name = "rejection_reason", columnDefinition = "TEXT")
    private String rejectionReason;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "verified_by")
    private User verifiedBy;
    
    @Column(name = "verified_at")
    private LocalDateTime verifiedAt;
    
    @Column(name = "expires_at")
    private LocalDateTime expiresAt;
    
    @Column(name = "metadata", columnDefinition = "TEXT")
    private String metadata;
    
    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
}
```

---

## API Endpoints

### 1. Register User

**Endpoint:** `POST /api/v1/auth/register`

**Request:**
```json
{
  "email": "student@fpt.edu.vn",
  "phone": "0901234567",
  "password": "SecurePass123!",
  "fullName": "Nguyễn Văn An",
  "dateOfBirth": "2000-01-15",
  "gender": "MALE"
}
```

**Response (201 Created):**
```json
{
  "user_id": 123,
  "user_type": "rider",
  "email": "student@fpt.edu.vn",
  "phone": "+84901234567",
  "full_name": "Nguyễn Văn An",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "created_at": "2025-12-04T10:30:00Z"
}
```

**Controller Code:**
```java
@RestController
@RequestMapping("/api/v1/auth")
@RequiredArgsConstructor
public class AuthController {
    
    private final AuthService authService;
    
    @PostMapping("/register")
    public ResponseEntity<RegisterResponse> register(
            @Valid @RequestBody RegisterRequest request) {
        
        log.info("Registration request for email: {}", request.getEmail());
        RegisterResponse response = authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

**Service Implementation:**
```java
@Service
@RequiredArgsConstructor
@Transactional
public class AuthServiceImpl implements AuthService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final OtpService otpService;
    private final EmailService emailService;
    private final JwtService jwtService;
    
    @Override
    public RegisterResponse register(RegisterRequest request) {
        // 1. Validate unique email
        if (userRepository.existsByEmail(request.getEmail())) {
            throw ConflictException.emailAlreadyExists(request.getEmail());
        }
        
        // 2. Normalize phone
        String normalizedPhone = PhoneUtil.normalize(request.getPhone());
        if (userRepository.existsByPhone(normalizedPhone)) {
            throw ConflictException.phoneAlreadyExists(request.getPhone());
        }
        
        // 3. Sanitize full name (prevent XSS)
        String sanitizedName = HtmlUtils.htmlEscape(request.getFullName());
        if (!ValidationUtil.isValidFullName(sanitizedName)) {
            throw ValidationException.invalidFullName();
        }
        
        // 4. Create user entity
        User user = User.builder()
                .email(request.getEmail())
                .phone(normalizedPhone)
                .passwordHash(passwordEncoder.encode(request.getPassword()))
                .fullName(sanitizedName)
                .dateOfBirth(request.getDateOfBirth())
                .gender(request.getGender())
                .userType(UserType.USER)
                .status(UserStatus.EMAIL_VERIFYING)
                .emailVerified(false)
                .phoneVerified(false)
                .build();
        
        user = userRepository.save(user);
        
        // 5. Generate and send OTP
        String otpCode = otpService.generateOtp(user, OtpFor.EMAIL_VERIFICATION);
        emailService.sendOtpEmail(user.getEmail(), user.getFullName(), otpCode);
        
        // 6. Generate JWT token
        Map<String, Object> claims = buildTokenClaims(user, null);
        String token = jwtService.generateToken(user.getEmail(), claims);
        
        // 7. Build response
        return RegisterResponse.builder()
                .userId(user.getUserId())
                .userType("rider")
                .email(user.getEmail())
                .phone(user.getPhone())
                .fullName(user.getFullName())
                .token(token)
                .createdAt(user.getCreatedAt())
                .build();
    }
}
```

---

### 2. Submit Student Verification

**Endpoint:** `POST /api/v1/me/student-verifications`

**Headers:**
```
Authorization: Bearer <JWT_TOKEN>
Content-Type: multipart/form-data
```

**Request (multipart form):**
- `document`: File[] (2 files - front and back)

**Response (201 Created):**
```json
{
  "verification_id": 456,
  "user_id": 123,
  "type": "STUDENT_ID",
  "status": "PENDING",
  "document_url": "https://s3.amazonaws.com/mssus/123_student_front.jpg,https://s3.amazonaws.com/mssus/123_student_back.jpg",
  "document_type": "IMAGE",
  "created_at": "2025-12-04T11:00:00Z"
}
```

**Controller Code:**
```java
@RestController
@RequestMapping("/api/v1/me")
@RequiredArgsConstructor
public class ProfileController {
    
    private final ProfileService profileService;
    
    @PreAuthorize("hasRole('USER')")
    @PostMapping(value = "/student-verifications", 
                 consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<VerificationResponse> submitStudentVerification(
            Authentication authentication,
            @RequestParam("document") List<MultipartFile> documents) {
        
        String username = authentication.getName();
        VerificationResponse response = profileService.submitStudentVerification(
            username, documents
        );
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

**Service Implementation:**
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProfileServiceImpl implements ProfileService {
    
    private final UserRepository userRepository;
    private final VerificationRepository verificationRepository;
    private final FileUploadService fileUploadService;
    private final VerificationMapper verificationMapper;
    
    @Override
    @Transactional
    public VerificationResponse submitStudentVerification(
            String username, List<MultipartFile> documents) {
        
        // 1. Find user
        User user = userRepository.findByEmail(username)
                .orElseThrow(() -> NotFoundException.userNotFound(username));
        
        // 2. Check email verified
        if (!user.getEmailVerified()) {
            throw ValidationException.of("Email must be verified first");
        }
        
        // 3. Validate documents
        if (documents == null || documents.isEmpty()) {
            throw ValidationException.of("At least one document required");
        }
        
        // 4. Check pending verification exists
        if (verificationRepository.findByUserIdAndTypeAndStatus(
                user.getUserId(), 
                VerificationType.STUDENT_ID, 
                VerificationStatus.PENDING
            ).isPresent()) {
            throw ConflictException.of("Pending verification already exists");
        }
        
        // 5. Check already verified
        if (verificationRepository.isUserVerifiedForType(
                user.getUserId(), VerificationType.STUDENT_ID)) {
            throw ConflictException.of("Student verification already approved");
        }
        
        try {
            // 6. Upload files in parallel
            List<CompletableFuture<String>> uploadFutures = documents.stream()
                    .map(file -> fileUploadService.uploadFile(file))
                    .collect(Collectors.toList());
            
            List<String> documentUrls = uploadFutures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.toList());
            
            String documentUrlsCombined = String.join(",", documentUrls);
            
            // 7. Create verification record
            Verification verification = Verification.builder()
                    .user(user)
                    .type(VerificationType.STUDENT_ID)
                    .status(VerificationStatus.PENDING)
                    .documentUrl(documentUrlsCombined)
                    .documentType(DocumentType.IMAGE)
                    .build();
            
            verification = verificationRepository.save(verification);
            
            // 8. Return response
            return verificationMapper.mapToVerificationResponse(verification);
            
        } catch (Exception e) {
            log.error("Failed to upload documents: {}", e.getMessage());
            throw new RuntimeException("Unable to upload documents: " + e.getMessage());
        }
    }
}
```

---

[← Quay lại tổng quan]({{ site.baseurl }}/flows/account-verification-activation/) | [Tiếp tục đọc API Endpoints →]({{ site.baseurl }}/flows/backend-implementation-api/)
