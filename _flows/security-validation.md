---
layout: default
title: Security & Validation
permalink: /flows/security-validation/
---

# Security & Validation

[← Quay lại tổng quan](../account-verification-activation/)

---

## 📋 Nội dung

Phần này giải thích các cơ chế bảo mật và validation trong Account Verification Flow:
- JWT Authentication
- Authorization với Spring Security
- Input validation
- File upload security
- XSS, SQL Injection prevention
- CSRF protection
- Rate limiting

---

## 1. JWT Authentication

### Token Structure

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.  ← Header
eyJ1c2VyX2lkIjoxMjMsImVtYWlsIjoic3R1ZGV...  ← Payload
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature
```

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (Claims):**
```json
{
  "user_id": 123,
  "email": "student@fpt.edu.vn",
  "full_name": "Nguyễn Văn An",
  "roles": ["ROLE_USER", "ROLE_RIDER"],
  "active_profile": "rider",
  "token_version": 1,
  "iat": 1701704800,
  "exp": 1701791200
}
```

### Token Generation

```java
@Service
public class JwtService {
    
    @Value("${jwt.secret}")
    private String secret; // 256-bit secret key
    
    @Value("${jwt.expiration}")
    private long expiration; // 24 hours = 86400000 ms
    
    public String generateToken(String email, Map<String, Object> claims) {
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(email)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(SignatureAlgorithm.HS256, secret)
                .compact();
    }
    
    public Claims extractClaims(String token) {
        return Jwts.parser()
                .setSigningKey(secret)
                .parseClaimsJws(token)
                .getBody();
    }
    
    public boolean isTokenValid(String token, UserDetails userDetails) {
        String email = extractClaims(token).getSubject();
        Date expiration = extractClaims(token).getExpiration();
        
        return email.equals(userDetails.getUsername()) && 
               !expiration.before(new Date());
    }
}
```

### JWT Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
        
        // 1. Extract token from Authorization header
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        String token = authHeader.substring(7);
        
        try {
            // 2. Extract email from token
            String email = jwtService.extractClaims(token).getSubject();
            
            // 3. Load user details
            UserDetails userDetails = userDetailsService.loadUserByUsername(email);
            
            // 4. Validate token
            if (jwtService.isTokenValid(token, userDetails)) {
                // 5. Create authentication
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities()
                    );
                
                // 6. Set authentication in SecurityContext
                SecurityContextHolder.getContext()
                    .setAuthentication(authentication);
            }
            
        } catch (JwtException | IllegalArgumentException e) {
            log.error("JWT validation failed: {}", e.getMessage());
        }
        
        filterChain.doFilter(request, response);
    }
}
```

---

## 2. Authorization với Spring Security

### Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthFilter;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) 
            throws Exception {
        
        http
            .csrf(csrf -> csrf.disable()) // Disable CSRF for stateless API
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/v1/auth/register", "/api/v1/auth/login")
                    .permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**")
                    .permitAll()
                    
                // User endpoints - require USER role
                .requestMatchers("/api/v1/me/**")
                    .hasRole("USER")
                    
                // Admin endpoints - require ADMIN role
                .requestMatchers("/api/v1/verification/**")
                    .hasRole("ADMIN")
                    
                // All other requests must be authenticated
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(unauthorizedHandler())
                .accessDeniedHandler(accessDeniedHandler())
            );
        
        return http.build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList(
            "http://localhost:3000", 
            "https://mssus-frontend.vercel.app"
        ));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "PATCH"));
        config.setAllowedHeaders(Arrays.asList("*"));
        config.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

### Method-level Security

```java
@RestController
@RequestMapping("/api/v1/me")
public class ProfileController {
    
    // Only users với ROLE_USER mới access được
    @PreAuthorize("hasRole('USER')")
    @PostMapping("/student-verifications")
    public ResponseEntity<VerificationResponse> submitStudentVerification(...) {
        // Implementation
    }
    
    // Only users với ROLE_RIDER mới access được
    @PreAuthorize("hasRole('RIDER')")
    @PostMapping("/driver-verifications/vehicle-registration")
    public ResponseEntity<VerificationResponse> submitVehicleRegistration(...) {
        // Implementation
    }
}

@RestController
@RequestMapping("/api/v1/verification")
public class VerificationController {
    
    // Chỉ ADMIN mới approve/reject được
    @PreAuthorize("hasRole('ADMIN')")
    @PostMapping("/approve")
    public ResponseEntity<MessageResponse> approveVerification(...) {
        // Implementation
    }
}
```

---

## 3. Input Validation

### Request DTO Validation

```java
@Data
@Builder
public class RegisterRequest {
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    @Size(max = 255)
    private String email;
    
    @NotBlank(message = "Phone is required")
    @Pattern(regexp = "^(\\+84|0)[0-9]{9,10}$", 
             message = "Phone number must be valid Vietnamese format")
    private String phone;
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 100)
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&]).*$",
             message = "Password must contain uppercase, lowercase, digit, and special char")
    private String password;
    
    @NotBlank(message = "Full name is required")
    @Size(min = 3, max = 100)
    @Pattern(regexp = "^[a-zA-ZÀ-ỹ\\s-]+$",
             message = "Full name can only contain letters, spaces, and hyphens")
    private String fullName;
    
    @Past(message = "Date of birth must be in the past")
    private LocalDate dateOfBirth;
}
```

### Validation Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException ex) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        ErrorResponse response = ErrorResponse.builder()
                .status(HttpStatus.BAD_REQUEST.value())
                .message("Validation failed")
                .errors(errors)
                .timestamp(LocalDateTime.now())
                .build();
        
        return ResponseEntity.badRequest().body(response);
    }
}
```

---

## 4. File Upload Security

### File Validation

```java
@Service
public class FileUploadService {
    
    private static final List<String> ALLOWED_CONTENT_TYPES = Arrays.asList(
        "image/jpeg",
        "image/png",
        "application/pdf"
    );
    
    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
    
    private static final int MIN_IMAGE_WIDTH = 800;
    private static final int MIN_IMAGE_HEIGHT = 600;
    
    public CompletableFuture<String> uploadFile(MultipartFile file) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                // 1. Validate file is not empty
                if (file.isEmpty()) {
                    throw new ValidationException("File is empty");
                }
                
                // 2. Validate content type
                String contentType = file.getContentType();
                if (!ALLOWED_CONTENT_TYPES.contains(contentType)) {
                    throw new ValidationException(
                        "File type not allowed. Only JPG, PNG, PDF accepted"
                    );
                }
                
                // 3. Validate file size
                if (file.getSize() > MAX_FILE_SIZE) {
                    throw new ValidationException(
                        "File too large. Maximum 10MB"
                    );
                }
                
                // 4. Validate image resolution (if image)
                if (contentType.startsWith("image/")) {
                    validateImageResolution(file);
                }
                
                // 5. Validate file content matches extension
                validateFileContent(file);
                
                // 6. Generate safe filename
                String safeFilename = generateSafeFilename(file);
                
                // 7. Upload to Cloudinary
                return uploadToCloudinary(file, safeFilename);
                
            } catch (Exception e) {
                throw new FileUploadException(
                    "Failed to upload file: " + e.getMessage()
                );
            }
        });
    }
    
    private void validateImageResolution(MultipartFile file) throws IOException {
        BufferedImage image = ImageIO.read(file.getInputStream());
        if (image == null) {
            throw new ValidationException("Invalid image file");
        }
        
        int width = image.getWidth();
        int height = image.getHeight();
        
        if (width < MIN_IMAGE_WIDTH || height < MIN_IMAGE_HEIGHT) {
            throw new ValidationException(
                String.format("Image resolution too low. Minimum %dx%d required",
                    MIN_IMAGE_WIDTH, MIN_IMAGE_HEIGHT)
            );
        }
    }
    
    private void validateFileContent(MultipartFile file) throws IOException {
        // Check magic bytes to prevent fake extensions
        byte[] fileBytes = new byte[8];
        file.getInputStream().read(fileBytes);
        
        String contentType = file.getContentType();
        if (contentType.equals("image/jpeg")) {
            // JPEG magic bytes: FF D8 FF
            if (fileBytes[0] != (byte) 0xFF || 
                fileBytes[1] != (byte) 0xD8 || 
                fileBytes[2] != (byte) 0xFF) {
                throw new ValidationException("File is not a valid JPEG");
            }
        } else if (contentType.equals("image/png")) {
            // PNG magic bytes: 89 50 4E 47
            if (fileBytes[0] != (byte) 0x89 || 
                fileBytes[1] != (byte) 0x50 || 
                fileBytes[2] != (byte) 0x4E || 
                fileBytes[3] != (byte) 0x47) {
                throw new ValidationException("File is not a valid PNG");
            }
        }
    }
    
    private String generateSafeFilename(MultipartFile file) {
        String originalFilename = file.getOriginalFilename();
        String extension = FilenameUtils.getExtension(originalFilename);
        
        // Generate UUID to prevent path traversal
        String uniqueId = UUID.randomUUID().toString();
        long timestamp = System.currentTimeMillis();
        
        return String.format("%d_%s.%s", timestamp, uniqueId, extension);
    }
}
```

---

## 5. XSS & SQL Injection Prevention

### XSS Prevention

```java
// HTML Escape user input
String sanitizedName = HtmlUtils.htmlEscape(request.getFullName());

// Example:
// Input: <script>alert('xss')</script>
// Output: &lt;script&gt;alert('xss')&lt;/script&gt;
```

### SQL Injection Prevention

```java
// ✅ GOOD - Sử dụng JPA/Hibernate với parameterized queries
@Repository
public interface UserRepository extends JpaRepository<User, Integer> {
    
    Optional<User> findByEmail(String email); // Safe - parameterized
    
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailCustom(@Param("email") String email); // Safe
}

// ❌ BAD - Không bao giờ concat string vào query
@Query(value = "SELECT * FROM users WHERE email = '" + email + "'", 
       nativeQuery = true) // VULNERABLE!
User findByEmailUnsafe(String email);
```

---

## 6. Rate Limiting

### Login Attempt Limiting

```java
@Service
@RequiredArgsConstructor
public class LoginAttemptService {
    
    private final ConcurrentHashMap<String, LoginAttempt> attempts = 
        new ConcurrentHashMap<>();
    
    private static final int MAX_ATTEMPTS = 5;
    private static final long LOCK_DURATION_MINUTES = 30;
    
    public void incrementFailedAttempts(String email) {
        LoginAttempt attempt = attempts.computeIfAbsent(email, 
            k -> new LoginAttempt());
        attempt.incrementFailed();
        
        if (attempt.getFailedAttempts() >= MAX_ATTEMPTS) {
            attempt.setLockTime(LocalDateTime.now().plusMinutes(LOCK_DURATION_MINUTES));
        }
    }
    
    public void resetFailedAttempts(String email) {
        attempts.remove(email);
    }
    
    public boolean isBlocked(String email) {
        LoginAttempt attempt = attempts.get(email);
        if (attempt == null) {
            return false;
        }
        
        if (attempt.getLockTime() != null && 
            LocalDateTime.now().isBefore(attempt.getLockTime())) {
            return true;
        }
        
        // Lock expired, reset
        if (attempt.getLockTime() != null) {
            attempts.remove(email);
        }
        
        return false;
    }
}
```

---

## 7. Password Security

### Password Hashing với BCrypt

```java
@Configuration
public class PasswordEncoderConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Strength 12
    }
}

// Usage
String hashedPassword = passwordEncoder.encode(plainPassword);
boolean matches = passwordEncoder.matches(plainPassword, hashedPassword);
```

**BCrypt Properties:**
- Salt tự động
- Adaptive (có thể tăng rounds khi CPU mạnh hơn)
- Slow by design (chống brute-force)

---

## 8. Token Revocation

### Token Version Strategy

```java
// Khi user change password, increment token_version
@Override
@Transactional
public void changePassword(Integer userId, String newPassword) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new NotFoundException("User not found"));
    
    // Update password
    user.setPasswordHash(passwordEncoder.encode(newPassword));
    
    // Increment token version → invalidate all existing tokens
    user.setTokenVersion(user.getTokenVersion() + 1);
    
    userRepository.save(user);
}

// Trong JWT filter, check token_version
public boolean isTokenValid(String token, UserDetails userDetails) {
    Claims claims = extractClaims(token);
    Integer tokenVersion = claims.get("token_version", Integer.class);
    
    User user = userRepository.findByEmail(userDetails.getUsername())
            .orElse(null);
    
    if (user == null || !tokenVersion.equals(user.getTokenVersion())) {
        return false; // Token revoked
    }
    
    return !claims.getExpiration().before(new Date());
}
```

---

[← Quay lại tổng quan](../account-verification-activation/) | [Đọc tiếp: Câu hỏi phản biện →](../defense-questions/)
