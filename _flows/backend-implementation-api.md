---
layout: default
title: Backend API - Admin Verification
permalink: /flows/account-verification-activation/backend-implementation-api/
---

# Backend API - Admin Verification Endpoints

[← Quay lại Backend Implementation](../backend-implementation)

---

## 3. Admin View All Verifications

**Endpoint:** `GET /api/v1/verification`

**Headers:**
```
Authorization: Bearer <ADMIN_JWT_TOKEN>
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | Integer | No | 0 | Page number (0-indexed) |
| `size` | Integer | No | 20 | Items per page |
| `sortBy` | String | No | `createdAt` | Sort field |
| `sortDir` | String | No | `desc` | `asc` or `desc` |

**Response (200 OK):**
```json
{
  "content": [
    {
      "verification_id": 456,
      "user_id": 123,
      "user": {
        "user_id": 123,
        "full_name": "Nguyễn Văn An",
        "email": "student@fpt.edu.vn",
        "phone": "+84901234567"
      },
      "type": "STUDENT_ID",
      "status": "PENDING",
      "document_url": "https://s3.../front.jpg,https://s3.../back.jpg",
      "document_type": "IMAGE",
      "created_at": "2025-12-04T11:00:00Z"
    }
  ],
  "page": 0,
  "page_size": 20,
  "total_pages": 5,
  "total_records": 95
}
```

**Controller:**
```java
@RestController
@RequestMapping("/api/v1/verification")
@RequiredArgsConstructor
public class VerificationController {
    
    private final VerificationService verificationService;
    
    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping
    public ResponseEntity<PageResponse<VerificationResponse>> getAllVerifications(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "desc") String sortDir) {
        
        Pageable pageable = PageRequest.of(
            page, size, 
            Sort.by(Sort.Direction.fromString(sortDir), sortBy)
        );
        
        PageResponse<VerificationResponse> response = 
            verificationService.getAllVerifications(pageable);
            
        return ResponseEntity.ok(response);
    }
}
```

**Service:**
```java
@Service
@RequiredArgsConstructor
public class VerificationServiceImpl implements VerificationService {
    
    private final VerificationRepository verificationRepository;
    private final VerificationMapper verificationMapper;
    
    @Override
    @Transactional(readOnly = true)
    public PageResponse<VerificationResponse> getAllVerifications(Pageable pageable) {
        Page<Verification> verificationPage = verificationRepository.findAll(pageable);
        
        List<VerificationResponse> content = verificationPage.getContent()
                .stream()
                .map(verificationMapper::mapToVerificationResponse)
                .collect(Collectors.toList());
        
        return PageResponse.<VerificationResponse>builder()
                .content(content)
                .page(verificationPage.getNumber())
                .pageSize(verificationPage.getSize())
                .totalPages(verificationPage.getTotalPages())
                .totalRecords(verificationPage.getTotalElements())
                .build();
    }
}
```

---

## 4. Admin Approve Verification

**Endpoint:** `POST /api/v1/verification/approve`

**Headers:**
```
Authorization: Bearer <ADMIN_JWT_TOKEN>
Content-Type: application/json
```

**Request:**
```json
{
  "userId": 123,
  "verificationType": "STUDENT_ID",
  "notes": "Giấy tờ hợp lệ, thông tin khớp"
}
```

**Response (200 OK):**
```json
{
  "message": "student_id approved successfully"
}
```

**Controller:**
```java
@PostMapping("/approve")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<MessageResponse> approveVerification(
        Authentication authentication,
        @Valid @RequestBody VerificationDecisionRequest request) {
    
    MessageResponse response = verificationService.approveVerification(
        authentication.getName(), request
    );
    return ResponseEntity.ok(response);
}
```

**Service Implementation:**
```java
@Override
@Transactional
public MessageResponse approveVerification(
        String adminEmail, VerificationDecisionRequest request) {
    
    // 1. Find user
    User user = userRepository.findById(request.getUserId())
            .orElseThrow(() -> new NotFoundException("User not found"));
    
    // 2. Find verification
    VerificationType type = VerificationType.valueOf(
        request.getVerificationType().toUpperCase()
    );
    Verification verification = verificationRepository
            .findByUserIdAndTypeAndStatus(
                user.getUserId(), type, VerificationStatus.PENDING
            )
            .orElseThrow(() -> new NotFoundException(
                type + " verification not found for user: " + user.getUserId()
            ));
    
    // 3. Find admin
    User admin = userRepository.findByEmail(adminEmail)
            .orElseThrow(() -> new NotFoundException("Admin not found"));
    
    // 4. Update verification
    verification.setStatus(VerificationStatus.APPROVED);
    verification.setVerifiedBy(admin);
    verification.setVerifiedAt(LocalDateTime.now());
    if (request.getNotes() != null) {
        verification.setMetadata(request.getNotes());
    }
    
    // 5. Activate profile based on type
    if (type == VerificationType.STUDENT_ID) {
        // Activate rider profile
        RiderProfile rider = ensureRiderProfile(user);
        rider.setStatus(RiderProfileStatus.ACTIVE);
        rider.setActivatedAt(LocalDateTime.now());
        riderProfileRepository.save(rider);
        
        // Create wallet if not exists
        if (user.getWallet() == null) {
            walletService.createWalletForUser(user.getUserId());
        }
        
        // Update user status
        user.setStatus(UserStatus.ACTIVE);
        userRepository.save(user);
        
        // Send activation email
        try {
            emailService.notifyRiderActivated(user);
        } catch (Exception e) {
            log.warn("Failed to send activation email: {}", e.getMessage());
        }
        
    } else if (isDriverVerification(type)) {
        // Mark driver as INACTIVE (waiting for more documents)
        DriverProfile driver = user.getDriverProfile();
        driver.setStatus(DriverProfileStatus.INACTIVE);
        driverProfileRepository.save(driver);
    }
    
    verificationRepository.save(verification);
    
    return MessageResponse.builder()
            .message(type.name().toLowerCase().replace("_", " ") + 
                    " approved successfully")
            .build();
}

private RiderProfile ensureRiderProfile(User user) {
    if (user.getRiderProfile() == null) {
        RiderProfile rider = RiderProfile.builder()
                .user(user)
                .status(RiderProfileStatus.PENDING)
                .build();
        return riderProfileRepository.save(rider);
    }
    return user.getRiderProfile();
}

private boolean isDriverVerification(VerificationType type) {
    return type == VerificationType.DRIVER_DOCUMENTS ||
           type == VerificationType.DRIVER_LICENSE ||
           type == VerificationType.VEHICLE_REGISTRATION ||
           type == VerificationType.BACKGROUND_CHECK;
}
```

---

## 5. Admin Reject Verification

**Endpoint:** `POST /api/v1/verification/reject`

**Request:**
```json
{
  "userId": 123,
  "verificationType": "STUDENT_ID",
  "rejectionReason": "Ảnh mờ, không rõ thông tin. Vui lòng chụp lại"
}
```

**Response (200 OK):**
```json
{
  "message": "Verification rejected"
}
```

**Service Implementation:**
```java
@Override
@Transactional
public MessageResponse rejectVerification(
        String adminEmail, VerificationDecisionRequest request) {
    
    // 1. Validate rejection reason
    if (request.getRejectionReason() == null || 
        request.getRejectionReason().trim().isEmpty()) {
        throw new ValidationException("Rejection reason is required");
    }
    
    // 2. Find verification
    User user = userRepository.findById(request.getUserId())
            .orElseThrow(() -> new NotFoundException("User not found"));
    
    VerificationType type = VerificationType.valueOf(
        request.getVerificationType().toUpperCase()
    );
    Verification verification = verificationRepository
            .findByUserIdAndTypeAndStatus(
                user.getUserId(), type, VerificationStatus.PENDING
            )
            .orElseThrow(() -> new NotFoundException("Verification not found"));
    
    // 3. Find admin
    User admin = userRepository.findByEmail(adminEmail)
            .orElseThrow(() -> new NotFoundException("Admin not found"));
    
    // 4. Update verification
    verification.setStatus(VerificationStatus.REJECTED);
    verification.setRejectionReason(request.getRejectionReason());
    verification.setVerifiedBy(admin);
    verification.setVerifiedAt(LocalDateTime.now());
    if (request.getNotes() != null) {
        verification.setMetadata(request.getNotes());
    }
    verificationRepository.save(verification);
    
    // 5. Update profile status
    if (user.getRiderProfile() != null && type == VerificationType.STUDENT_ID) {
        RiderProfile rider = user.getRiderProfile();
        rider.setStatus(RiderProfileStatus.REJECTED);
        riderProfileRepository.save(rider);
    } else if (user.getDriverProfile() != null && isDriverVerification(type)) {
        DriverProfile driver = user.getDriverProfile();
        driver.setStatus(DriverProfileStatus.REJECTED);
        driverProfileRepository.save(driver);
    }
    
    // 6. Send rejection email
    emailService.notifyUserRejected(user, type, request.getRejectionReason());
    
    log.warn("Verification {} rejected for user {}: {}", 
             type, user.getUserId(), request.getRejectionReason());
    
    return MessageResponse.builder()
            .message("Verification rejected")
            .build();
}
```

---

## 6. User Login

**Endpoint:** `POST /api/v1/auth/login`

**Request:**
```json
{
  "email": "student@fpt.edu.vn",
  "password": "SecurePass123!"
}
```

**Response (200 OK):**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 86400,
  "user": {
    "user_id": 123,
    "email": "student@fpt.edu.vn",
    "full_name": "Nguyễn Văn An",
    "user_type": "USER",
    "status": "ACTIVE",
    "roles": ["ROLE_USER", "ROLE_RIDER"],
    "active_profile": "rider"
  }
}
```

**Service Implementation:**
```java
@Override
public LoginResponse login(LoginRequest request) {
    // 1. Find user
    User user = userRepository.findByEmail(request.getEmail())
            .orElseThrow(() -> new UnauthorizedException("Invalid email or password"));
    
    // 2. Check password
    if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
        // Increment failed attempts
        loginAttemptService.incrementFailedAttempts(request.getEmail());
        throw new UnauthorizedException("Invalid email or password");
    }
    
    // 3. Check account status
    validateUserBeforeGrantingToken(user);
    
    // 4. Determine active profile
    String activeProfile = determineActiveProfile(user);
    
    // 5. Generate tokens
    Map<String, Object> claims = buildTokenClaims(user, activeProfile);
    String accessToken = jwtService.generateToken(user.getEmail(), claims);
    String refreshToken = jwtService.generateRefreshToken(user.getEmail());
    
    // 6. Save refresh token
    saveRefreshToken(user, refreshToken);
    
    // 7. Reset failed attempts
    loginAttemptService.resetFailedAttempts(request.getEmail());
    
    // 8. Build response
    return LoginResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .tokenType("Bearer")
            .expiresIn(jwtService.getExpirationTime())
            .user(userMapper.mapToUserResponse(user, activeProfile))
            .build();
}

private void validateUserBeforeGrantingToken(User user) {
    switch (user.getStatus()) {
        case EMAIL_VERIFYING:
            throw new UnauthorizedException("Please verify your email first");
        case PENDING:
            throw new UnauthorizedException(
                "Your account is pending verification. " +
                "Please upload identity documents or wait for admin approval"
            );
        case SUSPENDED:
            throw new UnauthorizedException(
                "Your account has been suspended. Contact support for assistance"
            );
        case BANNED:
            throw new UnauthorizedException("Your account has been banned");
        case ACTIVE:
            // OK
            break;
        default:
            throw new UnauthorizedException("Account not active");
    }
}

private String determineActiveProfile(User user) {
    // Priority: ADMIN > DRIVER (if active) > RIDER
    if (user.getUserType() == UserType.ADMIN) {
        return "admin";
    }
    
    if (user.getDriverProfile() != null && 
        user.getDriverProfile().getStatus() == DriverProfileStatus.ACTIVE) {
        return "driver";
    }
    
    if (user.getRiderProfile() != null && 
        user.getRiderProfile().getStatus() == RiderProfileStatus.ACTIVE) {
        return "rider";
    }
    
    return "rider"; // Default
}

public Map<String, Object> buildTokenClaims(User user, String activeProfile) {
    List<String> roles = new ArrayList<>();
    roles.add("ROLE_USER");
    
    if (user.getUserType() == UserType.ADMIN) {
        roles.add("ROLE_ADMIN");
    }
    if (user.getRiderProfile() != null && 
        user.getRiderProfile().getStatus() == RiderProfileStatus.ACTIVE) {
        roles.add("ROLE_RIDER");
    }
    if (user.getDriverProfile() != null && 
        user.getDriverProfile().getStatus() == DriverProfileStatus.ACTIVE) {
        roles.add("ROLE_DRIVER");
    }
    
    return Map.of(
        "user_id", user.getUserId(),
        "email", user.getEmail(),
        "full_name", user.getFullName(),
        "roles", roles,
        "active_profile", activeProfile != null ? activeProfile : "rider",
        "token_version", user.getTokenVersion()
    );
}
```

---

## Email Service Implementation

### Send OTP Email

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EmailServiceImpl implements EmailService {
    
    private final JavaMailSender mailSender;
    private final TemplateEngine templateEngine;
    
    @Value("${app.mail.from}")
    private String fromAddress;
    
    @Value("${app.mail.from-name}")
    private String fromName;
    
    @Override
    public CompletableFuture<EmailResult> sendOtpEmail(
            String toEmail, String fullName, String otpCode) {
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                Context context = new Context();
                context.setVariable("fullName", fullName);
                context.setVariable("otpCode", otpCode);
                context.setVariable("expiryMinutes", 10);
                
                String htmlContent = templateEngine.process(
                    "emails/otp-verification", context
                );
                
                MimeMessage message = mailSender.createMimeMessage();
                MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
                helper.setFrom(fromAddress, fromName);
                helper.setTo(toEmail);
                helper.setSubject("Xác thực email đăng ký tài khoản MSSUS");
                helper.setText(htmlContent, true);
                
                mailSender.send(message);
                
                log.info("OTP email sent to: {}", toEmail);
                return EmailResult.success("OTP email sent successfully");
                
            } catch (Exception e) {
                log.error("Failed to send OTP email to: {}", toEmail, e);
                return EmailResult.failure("Failed to send OTP email: " + e.getMessage());
            }
        });
    }
    
    @Override
    public CompletableFuture<EmailResult> notifyRiderActivated(User user) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                Context context = new Context();
                context.setVariable("fullName", user.getFullName());
                context.setVariable("email", user.getEmail());
                context.setVariable("approvalDate", 
                    LocalDateTime.now().format(DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm")));
                context.setVariable("frontendUrl", frontendBaseUrl);
                
                String htmlContent = templateEngine.process(
                    "emails/rider-verification-approved", context
                );
                
                MimeMessage message = mailSender.createMimeMessage();
                MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
                helper.setFrom(fromAddress, fromName);
                helper.setTo(user.getEmail());
                helper.setSubject("[Motorbike Sharing] Tài khoản đã được kích hoạt");
                helper.setText(htmlContent, true);
                
                mailSender.send(message);
                
                return EmailResult.success("Activation email sent");
            } catch (Exception e) {
                log.error("Failed to send activation email: {}", e.getMessage());
                return EmailResult.failure(e.getMessage());
            }
        });
    }
}
```

---

[← Quay lại Backend Implementation](../backend-implementation) | [Đọc tiếp: Security & Validation →](../security-validation)
