---
layout: default
title: Defense Questions & Answers
---

# Câu hỏi phản biện và trả lời

[← Quay lại tổng quan](../)

---

## 📋 Giới thiệu

Phần này tổng hợp các câu hỏi phản biện có thể gặp khi bảo vệ Account Verification & Activation Flow. Mỗi câu hỏi đi kèm câu trả lời chi tiết với lý do kỹ thuật và nghiệp vụ.

---

## 1. Business Logic Questions

### Q1: Tại sao lại cần manual approval thay vì dùng AI/OCR tự động verify?

**Trả lời:**

**Lý do chính:**
- **Accuracy requirements**: Hệ thống motorbike sharing yêu cầu 99.9% accuracy vì liên quan tới legal liability. Nếu cho thuê xe cho người không có GPLX → vi phạm pháp luật
- **Complex document types**: Sinh viên nộp CCCD + thẻ sinh viên, driver nộp GPLX, vehicle registration. AI/OCR khó handle nhiều template khác nhau
- **Cost-effectiveness**: OCR API (Google Vision, AWS Textract) tốn ~$1.50/1000 documents. Manual review chỉ cần 1-2 admin → rẻ hơn khi scale nhỏ
- **Fraud detection**: Admin có thể detect photoshop, fake documents dễ hơn AI

**Trade-offs:**
- ✅ Advantages: High accuracy, detect fraud, legal compliance
- ❌ Disadvantages: Slower (vài giờ thay vì instant), cần human resource
- 📊 Acceptable: Vì user không cần activate ngay lập tức (không phải ride-hailing realtime)

**Future enhancement:** Hybrid approach - AI pre-screen (reject rõ ràng fake) → admin final check

---

### Q2: Tại sao không merge 2 steps (upload docs + email verification) thành 1?

**Trả lời:**

**Two-step registration advantages:**

1. **Reduce bounce rate:**
   - Step 1 (signup) chỉ cần email/phone/password → 30 giây → dễ complete
   - Nếu bắt upload docs ngay → 5-10 phút → user drop off

2. **Email verification priority:**
   - Verify email trước → chặn fake email sớm
   - Nếu email invalid → không cần review docs (waste admin time)

3. **Better UX:**
   - User có thể signup trên mobile
   - Upload docs về sau khi có laptop (camera tốt hơn, file dễ manage)

**Alternative considered:**
- ❌ One-step signup: Upload docs luôn → too overwhelming → high bounce rate
- ❌ No email verification: Spammer tạo nhiều account fake → waste admin time

---

### Q3: Rejection reasons có thể được user xem lại để resubmit không?

**Trả lời:**

**Có - với retry mechanism:**

```sql
-- Database cho phép multiple verification attempts
ALTER TABLE verifications 
ADD COLUMN attempt_count INTEGER DEFAULT 1;
```

**Resubmit flow:**

1. Admin reject với reason: "Ảnh GPLX bị mờ, vui lòng chụp lại"
2. User nhận email notification với rejection reason
3. User login → dashboard show rejection reason
4. User có thể submit lại (POST `/me/driver-verifications/driver-license`)
5. Tạo verification record mới, `attempt_count++`

**Rate limiting:**
- Max 3 attempts/document type
- Nếu > 3 lần → require manual support (tránh abuse)

**Why this approach:**
- Giúp user self-service → reduce support burden
- Transparent feedback → improve document quality
- Track attempts → detect fraud patterns

---

### Q4: Làm thế nào đảm bảo admin không approve người không đủ điều kiện?

**Trả lời:**

**Multi-layer safeguards:**

1. **Admin training:**
   - Onboarding guide với document verification checklist
   - Examples: valid vs invalid documents
   - Monthly audit random samples

2. **Approval evidence:**
```java
@Entity
public class Verification {
    @Column(name = "reviewed_by_admin_id")
    private Integer reviewedByAdminId; // Trace who approved
    
    @Column(name = "review_notes")
    private String reviewNotes; // Admin notes (optional)
}
```

3. **Audit trail:**
   - Mỗi approval log trong database với timestamp + admin_id
   - Nếu có incident (e.g., xe bị đâm) → trace back admin approved fake GPLX

4. **Peer review (future):**
   - High-risk cases (e.g., GPLX mới cấp, foreign license) → require 2 admins

**Accountability:**
- Admin có username visible trong logs
- Nếu approve sai → internal review process

---

## 2. Technical Implementation Questions

### Q5: Tại sao dùng JWT thay vì session-based authentication?

**Trả lời:**

**JWT advantages cho stateless API:**

| Feature | JWT | Session |
|---------|-----|---------|
| **Scalability** | ✅ Stateless → dễ horizontal scale | ❌ Cần shared session store (Redis) |
| **Microservices** | ✅ Self-contained → không cần call auth service | ❌ Mỗi request check session DB |
| **Mobile app** | ✅ Token store dễ (localStorage) | ❌ Cookie không work tốt với mobile |
| **Performance** | ✅ Không cần DB query mỗi request | ❌ Mỗi request query session table |

**Trade-offs:**
- ❌ JWT cannot revoke (except token_version strategy)
- ❌ JWT payload visible (chỉ encode, không encrypt)

**Mitigation:**
- Token expiration: 24 hours
- Token version trong DB → revoke when change password
- HTTPS required → prevent token sniffing

---

### Q6: Database có index đúng chưa? Performance ra sao khi scale?

**Trả lời:**

**Critical indexes:**

```sql
-- Users table
CREATE INDEX idx_users_email ON users(email); -- Login lookup
CREATE INDEX idx_users_phone ON users(phone); -- Uniqueness check
CREATE INDEX idx_users_status ON users(status); -- Filter active users

-- Verifications table
CREATE INDEX idx_verifications_user_status 
    ON verifications(user_id, status); -- User's pending verifications
CREATE INDEX idx_verifications_status_created 
    ON verifications(status, created_at DESC); -- Admin queue
CREATE INDEX idx_verifications_type_status 
    ON verifications(verification_type, status); -- Filter by type

-- Profiles
CREATE INDEX idx_rider_profiles_user 
    ON rider_profiles(user_id); -- One-to-one lookup
CREATE INDEX idx_driver_profiles_user 
    ON driver_profiles(user_id);
```

**Query performance:**

```java
// Admin list pending verifications
// SELECT * FROM verifications 
// WHERE status = 'PENDING' 
// ORDER BY created_at DESC LIMIT 50;

// ✅ Uses index: idx_verifications_status_created
// Execution time: ~5ms for 100K records
```

**Scaling strategy:**
- Current: Single PostgreSQL instance
- 10K users: Add read replicas
- 100K users: Partition verifications table by created_at (monthly)
- 1M users: Separate verification service + database

---

### Q7: File upload có handle concurrent requests không? Nếu user upload 3 files cùng lúc?

**Trả lời:**

**Concurrency handling:**

Frontend gọi **parallel uploads** với `Promise.all()`:

```typescript
const uploadPromises = [
  uploadFile(studentIdFront, "STUDENT_ID_FRONT"),
  uploadFile(studentIdBack, "STUDENT_ID_BACK"),
  uploadFile(studentCard, "STUDENT_CARD")
];

const urls = await Promise.all(uploadPromises);
```

Backend handle **3 async uploads** với `CompletableFuture`:

```java
CompletableFuture<String> future1 = fileUploadService.uploadFile(file1);
CompletableFuture<String> future2 = fileUploadService.uploadFile(file2);
CompletableFuture<String> future3 = fileUploadService.uploadFile(file3);

CompletableFuture.allOf(future1, future2, future3).join();
```

**Database transaction:**
- Chỉ tạo 1 Verification record **sau khi** all files uploaded
- `@Transactional` → nếu 1 file fail → rollback all

**Rate limiting:**
- Max 3 concurrent uploads/user (prevent abuse)
- AWS S3 handles high throughput (11M requests/second)

---

### Q8: Email service có retry mechanism không? Nếu SMTP server down?

**Trả lời:**

**Current implementation:**

```java
@Async
public CompletableFuture<Void> sendActivationEmail(String to, String fullName) {
    try {
        MimeMessage message = mailSender.createMimeMessage();
        // ... send email
        mailSender.send(message);
        return CompletableFuture.completedFuture(null);
    } catch (MailException e) {
        log.error("Failed to send email to {}: {}", to, e.getMessage());
        throw new EmailException("Email send failed");
    }
}
```

**Problem:** No retry → nếu SMTP down → email lost

**Enhanced implementation (recommended):**

```java
@Retryable(
    value = {MailException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 2000, multiplier = 2)
)
public CompletableFuture<Void> sendActivationEmail(...) {
    // ... implementation
}

// Hoặc dùng message queue:
// User approved → publish message to RabbitMQ
// Email worker consume queue → retry with exponential backoff
```

**Alternative (production-ready):**
- Dùng SendGrid/AWS SES thay vì SMTP → 99.9% uptime
- Email queue table:
```sql
CREATE TABLE email_queue (
    id SERIAL PRIMARY KEY,
    to_email VARCHAR(255),
    subject VARCHAR(500),
    body TEXT,
    status VARCHAR(20), -- PENDING, SENT, FAILED
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP
);
```

---

### Q9: JWT secret key được store ở đâu? Có bảo mật không?

**Trả lời:**

**Current setup:**

```yaml
# application.yml (NOT in Git)
jwt:
  secret: ${JWT_SECRET:default-secret-key-change-in-production}
  expiration: 86400000 # 24 hours
```

**Environment variables:**

```bash
# Production (AWS EC2)
export JWT_SECRET="a7f9c3e1b4d8f2a6c9e5b1d3f7a2c4e8..."

# Docker
docker run -e JWT_SECRET="..." backend-app
```

**Best practices:**
- ✅ Secret key = 256-bit random string (64 hex chars)
- ✅ Different key per environment (dev/staging/prod)
- ✅ Stored in AWS Secrets Manager (production)
- ✅ Rotate every 90 days

**Key generation:**
```bash
openssl rand -hex 32
# Output: a7f9c3e1b4d8f2a6c9e5b1d3f7a2c4e8b3d9f1a7c5e2b8d4f6a9c3e1b5d7f2a4
```

**Security:**
- ❌ Never hardcode in source code
- ❌ Never commit to Git (use .env.example template)
- ✅ Use secret management service (AWS Secrets Manager, HashiCorp Vault)

---

### Q10: API có versioning không? Breaking changes sẽ handle như thế nào?

**Trả lời:**

**Current versioning:**

```java
@RestController
@RequestMapping("/api/v1/auth")
public class AuthController { ... }

@RestController
@RequestMapping("/api/v1/me")
public class ProfileController { ... }
```

**Versioning strategy:**
- URI versioning (`/api/v1`, `/api/v2`)
- Major version in path → easy for clients to see
- Maintain backward compatibility within same major version

**Breaking change example:**

```java
// v1 - current
POST /api/v1/me/student-verifications
Request: { studentIdFront, studentIdBack, studentCardUrl }

// v2 - future (separate endpoints)
POST /api/v2/me/documents/student-id
POST /api/v2/me/documents/student-card
```

**Migration plan:**
1. Release v2 alongside v1
2. Deprecation notice in v1 responses:
```json
{
  "data": {...},
  "deprecation": {
    "message": "v1 will be deprecated on 2025-06-01",
    "migrate_to": "https://docs.mssus.com/api/v2"
  }
}
```
3. Sunset v1 after 6 months

---

## 3. Architecture & Design Questions

### Q11: Tại sao không tách Verification thành microservice riêng?

**Trả lời:**

**Current architecture:** Monolithic Spring Boot

**Rationale:**
- **Team size:** 3-4 developers → microservices overhead cao (deployment, monitoring, debugging)
- **Business coupling:** Verification tightly coupled với User, Profile → nhiều inter-service calls → latency cao
- **Database transactions:** Approval cần update Users + Verifications + Profiles atomically → distributed transaction phức tạp
- **Deployment complexity:** 1 service dễ deploy hơn 5 services

**When to split:**
- Team > 10 người
- Verification có different scaling needs (e.g., AI/OCR service needs GPU)
- Verification được reuse bởi other systems

**Current mitigation:**
- Modular design: Clear package separation (`auth`, `verification`, `profile`)
- Interface-based: Easy to extract later nếu cần

---

### Q12: Có sử dụng caching không? Nếu có, cache strategy ra sao?

**Trả lời:**

**Current implementation:** No caching layer

**Should cache:**

1. **User profile data:**
```java
@Cacheable(value = "users", key = "#userId")
public User getUserById(Integer userId) {
    return userRepository.findById(userId)
            .orElseThrow(() -> new NotFoundException("User not found"));
}

@CacheEvict(value = "users", key = "#userId")
public void updateUser(Integer userId, UserUpdateRequest request) {
    // Update user → invalidate cache
}
```

2. **Verification counts (admin dashboard):**
```java
@Cacheable(value = "verification-stats", key = "'pending-count'", 
           unless = "#result == null")
public Long getPendingVerificationCount() {
    return verificationRepository.countByStatus(VerificationStatus.PENDING);
}
```

**Cache strategy:**
- **Cache store:** Redis (standalone or cluster)
- **TTL:** 5 minutes cho user profile, 1 minute cho stats
- **Eviction:** Write-through (update DB → invalidate cache)

**Trade-offs:**
- ✅ Reduce DB load (mỗi API call không query DB)
- ✅ Faster response time (Redis < 1ms vs PostgreSQL 5-10ms)
- ❌ Cache invalidation complexity
- ❌ Eventual consistency (cache có thể stale vài giây)

---

### Q13: Error handling có consistent không? HTTP status codes dùng đúng chưa?

**Trả lời:**

**Standardized error responses:**

```java
@Data
@Builder
public class ErrorResponse {
    private int status;
    private String message;
    private Map<String, String> errors; // Field-level errors
    private LocalDateTime timestamp;
    private String path;
}
```

**HTTP status codes:**

| Status | Use Case | Example |
|--------|----------|---------|
| 200 OK | Success | Login success, get verifications |
| 201 Created | Resource created | User registered, verification submitted |
| 400 Bad Request | Validation error | Invalid email format |
| 401 Unauthorized | Not authenticated | No JWT token |
| 403 Forbidden | Not authorized | User access admin endpoint |
| 404 Not Found | Resource not exist | User ID not found |
| 409 Conflict | Business rule violation | Email already exists |
| 500 Internal Error | Server error | Database connection failed |

**Example responses:**

```json
// 400 - Validation error
{
  "status": 400,
  "message": "Validation failed",
  "errors": {
    "email": "Email must be valid",
    "password": "Password must be at least 8 characters"
  },
  "timestamp": "2024-01-15T10:30:00",
  "path": "/api/v1/auth/register"
}

// 409 - Business conflict
{
  "status": 409,
  "message": "Email already registered",
  "timestamp": "2024-01-15T10:30:00",
  "path": "/api/v1/auth/register"
}
```

---

## 4. Security Questions

### Q14: Có test penetration/security vulnerabilities chưa?

**Trả lời:**

**Security measures implemented:**

1. **OWASP Top 10 coverage:**
   - ✅ SQL Injection: JPA parameterized queries
   - ✅ XSS: HTML escaping user input
   - ✅ Broken Authentication: JWT + BCrypt passwords
   - ✅ Sensitive Data Exposure: HTTPS only, no passwords in logs
   - ✅ Broken Access Control: Role-based authorization
   - ✅ Security Misconfiguration: CORS configured, CSRF disabled (stateless)

2. **Security testing:**
   - Dependency scanning: `mvn dependency-check:check` (OWASP Dependency-Check)
   - Static analysis: SonarQube (detect code smells, vulnerabilities)
   - Manual penetration testing: Basic (login brute-force, SQL injection attempts)

**Future improvements:**
- Automated security scans in CI/CD (Snyk, GitLab SAST)
- Professional penetration testing before production
- Bug bounty program

---

### Q15: Rate limiting có apply cho tất cả endpoints không?

**Trả lời:**

**Current state:** Rate limiting chỉ có cho login attempts

**Should add global rate limiting:**

```java
@Configuration
public class RateLimitConfig {
    
    @Bean
    public RateLimiter apiRateLimiter() {
        // 100 requests per minute per IP
        return RateLimiter.create(100.0 / 60.0); // requests/second
    }
}

@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    @Autowired
    private RateLimiter rateLimiter;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response,
                                    FilterChain filterChain) {
        if (!rateLimiter.tryAcquire()) {
            response.setStatus(429); // Too Many Requests
            response.getWriter().write("Rate limit exceeded");
            return;
        }
        filterChain.doFilter(request, response);
    }
}
```

**Different limits per endpoint:**
- `/auth/login`: 5 requests/minute
- `/auth/register`: 3 requests/hour
- `/me/*`: 60 requests/minute
- `/verification/*` (admin): 100 requests/minute

---

## 5. Performance & Scalability Questions

### Q16: Load testing results ra sao? System handle được bao nhiêu users?

**Trả lời:**

**Current capacity (single instance):**

| Endpoint | Avg Response Time | Throughput |
|----------|-------------------|------------|
| POST /login | 150ms | 50 req/s |
| GET /verification | 80ms | 100 req/s |
| POST /student-verifications | 2000ms | 10 req/s (file upload) |

**Bottlenecks:**
1. File upload: Limited by network bandwidth (AWS S3 upload 5-10 MB/s per connection)
2. Email sending: SMTP server limits (100 emails/minute)

**Scaling plan:**

| Users | Infrastructure | Expected Load |
|-------|---------------|---------------|
| 1K | Single EC2 t3.medium | 10-20 requests/minute |
| 10K | EC2 t3.large + RDS Multi-AZ | 100-200 requests/minute |
| 100K | Auto Scaling Group (3-5 instances) + CloudFront CDN | 1000+ requests/minute |

**Load testing tools:**
- JMeter scenarios: 1000 concurrent users, 10-minute ramp-up
- Gatling: HTTP request simulation với realistic user behavior

---

### Q17: Database migration (Flyway) có rollback mechanism không?

**Trả lời:**

**Current setup:**

```
backend/src/main/resources/db/migration/
├── V1__Create_users_table.sql
├── V2__Create_verifications_table.sql
├── V3__Create_rider_profiles_table.sql
└── V4__Create_driver_profiles_table.sql
```

**Flyway properties:**
```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    validate-on-migrate: true
```

**Rollback strategy:**

Flyway Community Edition **không support auto-rollback** (cần Flyway Teams)

**Manual rollback process:**

1. Create compensating migration:
```sql
-- V5__Add_user_rating_column.sql (WRONG - need rollback)
ALTER TABLE users ADD COLUMN rating DECIMAL(3,2);

-- V6__Rollback_user_rating.sql (Compensating migration)
ALTER TABLE users DROP COLUMN IF EXISTS rating;
```

2. Or restore from database backup:
```bash
pg_restore --clean --if-exists -d mssus backup_20240115.dump
```

**Best practices:**
- Test migrations in staging first
- Always backup before migration
- Keep migrations small (1 migration = 1 logical change)
- Use `IF EXISTS` / `IF NOT EXISTS` for idempotency

---

## 6. Alternative Approaches

### Q18: Có cân nhắc dùng OAuth2 (Google/Facebook login) không?

**Trả lời:**

**Current:** Email/password authentication only

**OAuth2 advantages:**
- ✅ Better UX (no password to remember)
- ✅ Higher security (Google/FB handle credentials)
- ✅ Faster signup (auto-fill name, email)

**Why not implemented:**
- ❌ Still need document verification → OAuth doesn't help with core flow
- ❌ Extra complexity (handle both email + OAuth users)
- ❌ Privacy concerns (users may not want to link FB account to motorbike rental)

**Future consideration:**
- Add OAuth as **optional** alongside email/password
- Use email as primary identifier (unified user table)

```java
@Entity
public class User {
    @Column(unique = true)
    private String email; // Primary identifier
    
    private String passwordHash; // Nullable (if OAuth)
    
    @Enumerated(EnumType.STRING)
    private AuthProvider authProvider; // EMAIL, GOOGLE, FACEBOOK
    
    private String oauthId; // Google/FB user ID
}
```

---

### Q19: Có thể dùng WebSocket cho real-time notification thay vì polling không?

**Trả lời:**

**Current:** Email notification only (one-way)

**WebSocket use case:**
- Admin approve → WebSocket push notification → user dashboard updates real-time

**Implementation:**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}

// Send notification
@Autowired
private SimpMessagingTemplate messagingTemplate;

public void notifyUserApproved(Integer userId) {
    messagingTemplate.convertAndSend(
        "/topic/user/" + userId,
        new NotificationMessage("Your account has been approved!")
    );
}
```

**Frontend:**
```typescript
const socket = new SockJS('http://localhost:8080/ws');
const stompClient = Stomp.over(socket);

stompClient.subscribe('/topic/user/' + userId, (message) => {
  showToast(message.body);
});
```

**Trade-offs:**
- ✅ Real-time updates (no page refresh)
- ❌ Maintain persistent connections (resource intensive)
- ❌ Scaling challenge (WebSocket sticky sessions)

**Current approach sufficient:** Email notification adequate vì user không cần instant update

---

### Q20: Tại sao không dùng GraphQL thay vì REST?

**Trả lời:**

**REST advantages cho use case này:**

1. **Simplicity:**
   - Endpoints rõ ràng: `POST /login`, `GET /verification`
   - Frontend developers dễ hiểu hơn

2. **Caching:**
   - HTTP caching (CDN, browser cache) work out-of-the-box
   - GraphQL caching phức tạp hơn

3. **File upload:**
   - REST multipart/form-data đơn giản
   - GraphQL file upload cần extra setup (Apollo Upload)

**GraphQL advantages (không critical cho project này):**
- Over-fetching: REST có thể return unnecessary fields
- Under-fetching: REST cần multiple requests cho related data

**Example:**

```
// REST - 2 requests
GET /users/123 → { id, name, email }
GET /users/123/verifications → [{ type, status, url }]

// GraphQL - 1 request
query {
  user(id: 123) {
    id
    name
    email
    verifications {
      type
      status
      url
    }
  }
}
```

**Verdict:** REST đủ cho MSSUS (simple data model, không có complex nested queries)

---

[← Quay lại tổng quan](../account-verification-activation/)
