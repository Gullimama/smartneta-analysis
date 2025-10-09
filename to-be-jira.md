# JIRA Stories: SmartNeta To-Be Architecture Implementation

**Epic:** Modernize SmartNeta Platform with Cloud-Native Architecture

**Timeline:** 12 months (4 quarters)  
**Team:** Architecture Lead, Backend Developers, Mobile Developers, QA Engineers, DevOps Engineers, UI/UX Designer

---

## Epic Overview

**Epic ID:** MODERNIZE-001  
**Epic Name:** Transform SmartNeta to Modern Cloud-Native Architecture  
**Business Value:** Achieve 5x performance improvement, 99.5% availability, zero security vulnerabilities, and 3x development velocity  
**Success Criteria:**
- Response time < 200ms (p95)
- Support 5,000+ concurrent users
- Zero critical security vulnerabilities
- 99.5% uptime
- Modern mobile PWA with offline-first capabilities

---

## Q1 2024: Critical Security & Framework Upgrades (Months 1-3)

### Priority 1: Security Vulnerabilities & Framework Upgrades

---

### Story 1: Backend Framework Security Upgrade

**Story ID:** MOD-101  
**Story Points:** 13  
**Priority:** Critical  
**Assignee:** Backend Developer + Security Architect  
**Quarter:** Q1

**User Story:**
As a backend developer, I need to upgrade Spring Boot from 2.0.3.RELEASE to 3.2+ with Java 17 LTS so that all critical security vulnerabilities are eliminated and modern framework features are available.

**Acceptance Criteria:**
- [ ] Audit current Spring Boot 2.0.3 dependencies and identify breaking changes
- [ ] Create migration plan with dependency mapping
- [ ] Upgrade to Java 17 LTS and update all Java code for compatibility
- [ ] Upgrade Spring Boot to 3.2+ with all dependencies
- [ ] Update Spring Data JPA to 3.2+ compatible version
- [ ] Update Hibernate to 6.x
- [ ] Migrate from javax.* to jakarta.* packages
- [ ] Update all configuration files (application.properties, beans)
- [ ] Fix all compilation errors and deprecation warnings
- [ ] Run full test suite and fix failing tests
- [ ] Perform security scan with Snyk/OWASP
- [ ] Load test to ensure no performance regression
- [ ] Document migration steps and gotchas

**Technical Details:**

**pom.xml updates:**
```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.2.0</spring-boot.version>
</properties>

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <!-- Spring Boot 3.2+ -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    
    <!-- Jakarta EE -->
    <dependency>
        <groupId>jakarta.persistence</groupId>
        <artifactId>jakarta.persistence-api</artifactId>
    </dependency>
    <dependency>
        <groupId>jakarta.validation</groupId>
        <artifactId>jakarta.validation-api</artifactId>
    </dependency>
</dependencies>
```

**Package Migration:**
```java
// Before (javax)
import javax.persistence.Entity;
import javax.validation.constraints.NotNull;

// After (jakarta)
import jakarta.persistence.Entity;
import jakarta.validation.constraints.NotNull;
```

**Testing:**
- Run all existing unit and integration tests
- Security scan: `mvn dependency-check:check`
- Load test with JMeter (baseline vs upgraded)

**Definition of Done:**
- Spring Boot upgraded to 3.2+
- Java 17 LTS in use
- All tests passing
- Security scan shows zero critical vulnerabilities
- No performance regression
- Migration guide documented
- Code review approved

---

### Story 2: Mobile Framework Security Upgrade

**Story ID:** MOD-102  
**Story Points:** 13  
**Priority:** Critical  
**Assignee:** Mobile Developer + Security Architect  
**Quarter:** Q1

**User Story:**
As a mobile developer, I need to upgrade Ionic from 3.9.10 to 7+ and Angular from 5.0.1 to 17+ so that all security vulnerabilities are eliminated and modern mobile features are available.

**Acceptance Criteria:**
- [ ] Audit current Ionic 3.9.10 and Angular 5.0.1 codebase
- [ ] Create new Ionic 7 project with Angular 17
- [ ] Migrate all pages/components to new project structure
- [ ] Update all Angular syntax (RxJS operators, lifecycle hooks)
- [ ] Migrate from Ionic 3 navigation to Ionic 7 routing
- [ ] Update all Cordova plugins to Capacitor plugins
- [ ] Update all UI components to Ionic 7 syntax
- [ ] Migrate from Angular HttpModule to HttpClient
- [ ] Update all TypeScript code to ES2022 standards
- [ ] Run full test suite with Jasmine/Karma
- [ ] Test on iOS and Android devices
- [ ] Perform security scan with Snyk
- [ ] Document migration steps

**Technical Details:**

**Create New Project:**
```bash
npm install -g @ionic/cli@latest
ionic start smartneta-mobile blank --type=angular
cd smartneta-mobile
npm install @angular/core@17 @angular/common@17
```

**package.json:**
```json
{
  "dependencies": {
    "@angular/animations": "^17.0.0",
    "@angular/common": "^17.0.0",
    "@angular/core": "^17.0.0",
    "@angular/forms": "^17.0.0",
    "@angular/platform-browser": "^17.0.0",
    "@angular/platform-browser-dynamic": "^17.0.0",
    "@angular/router": "^17.0.0",
    "@capacitor/android": "^5.0.0",
    "@capacitor/core": "^5.0.0",
    "@capacitor/ios": "^5.0.0",
    "@ionic/angular": "^7.0.0",
    "rxjs": "^7.8.0",
    "tslib": "^2.6.0",
    "zone.js": "^0.14.0"
  }
}
```

**Migration Examples:**

**Navigation (Ionic 3 → 7):**
```typescript
// Before (Ionic 3)
this.navCtrl.push(DetailsPage, { id: 123 });

// After (Ionic 7)
this.router.navigate(['/details', 123]);
```

**HTTP (Angular 5 → 17):**
```typescript
// Before (Angular 5)
import { Http } from '@angular/http';
this.http.get(url).map(res => res.json());

// After (Angular 17)
import { HttpClient } from '@angular/common/http';
this.http.get<T>(url).pipe(map(res => res));
```

**RxJS Operators:**
```typescript
// Before (RxJS 5)
import 'rxjs/add/operator/map';
observable.map(x => x);

// After (RxJS 7)
import { map } from 'rxjs/operators';
observable.pipe(map(x => x));
```

**Testing:**
```bash
# Unit tests
npm run test

# E2E tests
npm run e2e

# Build for production
ionic build --prod

# Test on devices
ionic capacitor run ios
ionic capacitor run android
```

**Definition of Done:**
- Ionic upgraded to 7+
- Angular upgraded to 17+
- All pages and components migrated
- All tests passing
- App builds and runs on iOS and Android
- Security scan shows zero critical vulnerabilities
- Migration guide documented
- Code review approved

---

### Story 3: Database Security Hardening

**Story ID:** MOD-103  
**Story Points:** 13  
**Priority:** Critical  
**Assignee:** Backend Developer + DBA  
**Quarter:** Q1

**User Story:**
As a DBA, I need to migrate from MySQL to PostgreSQL 15+ with encryption at rest and in transit so that data security is enhanced and modern database features are available.

**Acceptance Criteria:**
- [ ] Set up PostgreSQL 15+ instance with encryption
- [ ] Create database schema migration scripts
- [ ] Migrate all MySQL tables to PostgreSQL
- [ ] Update all SQL queries for PostgreSQL syntax
- [ ] Implement connection pooling with HikariCP
- [ ] Configure SSL/TLS for database connections
- [ ] Set up automated backups
- [ ] Implement point-in-time recovery
- [ ] Test data migration with production-like dataset
- [ ] Performance test PostgreSQL vs MySQL
- [ ] Document migration procedure
- [ ] Create rollback plan

**Technical Details:**

**PostgreSQL Setup:**
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: smartneta
      POSTGRES_USER: smartneta_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --data-checksums"
    command:
      - "postgres"
      - "-c"
      - "ssl=on"
      - "-c"
      - "ssl_cert_file=/var/lib/postgresql/server.crt"
      - "-c"
      - "ssl_key_file=/var/lib/postgresql/server.key"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./certs:/var/lib/postgresql/certs
    ports:
      - "5432:5432"
```

**application.properties:**
```properties
# PostgreSQL configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/smartneta?ssl=true&sslmode=require
spring.datasource.username=smartneta_user
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# HikariCP
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000

# JPA
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
```

**Migration Script (Flyway):**
```sql
-- V1__initial_schema.sql
CREATE TABLE citizen (
    id BIGSERIAL PRIMARY KEY,
    voter_id VARCHAR(50) UNIQUE NOT NULL,
    first_name VARCHAR(100),
    family_name VARCHAR(100),
    age INTEGER,
    gender VARCHAR(10),
    mobile VARCHAR(15),
    address TEXT,
    state VARCHAR(100),
    assembly_no VARCHAR(50),
    ward_no VARCHAR(50),
    booth_no VARCHAR(50),
    responded_status VARCHAR(50),
    voted BOOLEAN DEFAULT FALSE,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    modified_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_citizen_voter_id ON citizen(voter_id);
CREATE INDEX idx_citizen_location ON citizen(state, assembly_no, ward_no, booth_no);
CREATE INDEX idx_citizen_modified ON citizen(modified_date);
```

**Data Migration:**
```bash
# Export from MySQL
mysqldump -u root -p sampark > mysql_dump.sql

# Convert MySQL dump to PostgreSQL
pgloader mysql://root:password@localhost/sampark postgresql://smartneta_user:password@localhost/smartneta

# Verify data
psql -U smartneta_user -d smartneta -c "SELECT COUNT(*) FROM citizen;"
```

**Testing:**
- Migrate test dataset (100K records)
- Run all integration tests
- Performance comparison (query execution time)
- Verify data integrity

**Definition of Done:**
- PostgreSQL 15+ deployed with encryption
- All data migrated successfully
- All queries updated for PostgreSQL
- SSL/TLS enabled
- Automated backups configured
- Performance meets or exceeds MySQL
- Migration guide documented
- Rollback plan tested
- Code review approved

---

### Story 4: Authentication Security Overhaul

**Story ID:** MOD-104  
**Story Points:** 13  
**Priority:** Critical  
**Assignee:** Backend Developer + Security Architect  
**Quarter:** Q1

**User Story:**
As a security architect, I need to replace Apache Shiro 1.4.0-RC2 with Spring Security 6+ and OAuth2/JWT so that authentication is secure, modern, and standards-compliant.

**Acceptance Criteria:**
- [ ] Remove Apache Shiro dependencies
- [ ] Implement Spring Security 6+ with OAuth2
- [ ] Implement JWT token generation and validation
- [ ] Implement refresh token mechanism
- [ ] Implement role-based access control (RBAC)
- [ ] Implement password encryption with BCrypt
- [ ] Implement multi-factor authentication (MFA)
- [ ] Create authentication endpoints (/login, /logout, /refresh)
- [ ] Implement token blacklisting for logout
- [ ] Add security headers (CORS, CSP, X-Frame-Options)
- [ ] Test authentication flows
- [ ] Security audit with OWASP ZAP
- [ ] Document authentication architecture

**Technical Details:**

**pom.xml:**
```xml
<dependencies>
    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    
    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**SecurityConfig.java:**
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Autowired
    private JwtAuthenticationFilter jwtAuthFilter;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/volunteer/**").hasAnyRole("VOLUNTEER", "ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) 
            throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList("https://app.smartneta.com"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "PATCH"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

**JwtService.java:**
```java
@Service
public class JwtService {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private long jwtExpiration;
    
    @Value("${jwt.refresh-expiration}")
    private long refreshExpiration;
    
    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }
    
    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return buildToken(extraClaims, userDetails, jwtExpiration);
    }
    
    public String generateRefreshToken(UserDetails userDetails) {
        return buildToken(new HashMap<>(), userDetails, refreshExpiration);
    }
    
    private String buildToken(
            Map<String, Object> extraClaims,
            UserDetails userDetails,
            long expiration
    ) {
        return Jwts.builder()
                .claims(extraClaims)
                .subject(userDetails.getUsername())
                .issuedAt(new Date(System.currentTimeMillis()))
                .expiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSignInKey())
                .compact();
    }
    
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername())) && !isTokenExpired(token);
    }
    
    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
    
    private Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSignInKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
    
    private SecretKey getSignInKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

**AuthenticationController.java:**
```java
@RestController
@RequestMapping("/api/v1/auth")
public class AuthenticationController {
    
    @Autowired
    private AuthenticationService authService;
    
    @PostMapping("/login")
    public ResponseEntity<AuthenticationResponse> login(
            @Valid @RequestBody LoginRequest request
    ) {
        return ResponseEntity.ok(authService.authenticate(request));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<AuthenticationResponse> refresh(
            @Valid @RequestBody RefreshTokenRequest request
    ) {
        return ResponseEntity.ok(authService.refreshToken(request));
    }
    
    @PostMapping("/logout")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> logout(
            @RequestHeader("Authorization") String token
    ) {
        authService.logout(token);
        return ResponseEntity.ok().build();
    }
    
    @PostMapping("/forgot-password")
    public ResponseEntity<Void> forgotPassword(
            @Valid @RequestBody ForgotPasswordRequest request
    ) {
        authService.forgotPassword(request);
        return ResponseEntity.ok().build();
    }
    
    @PostMapping("/reset-password")
    public ResponseEntity<Void> resetPassword(
            @Valid @RequestBody ResetPasswordRequest request
    ) {
        authService.resetPassword(request);
        return ResponseEntity.ok().build();
    }
}
```

**Testing:**
```bash
# Login
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}'

# Access protected endpoint
curl -X GET http://localhost:8080/api/v1/users \
  -H "Authorization: Bearer <token>"

# Refresh token
curl -X POST http://localhost:8080/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"<refresh_token>"}'

# Logout
curl -X POST http://localhost:8080/api/v1/auth/logout \
  -H "Authorization: Bearer <token>"
```

**Security Testing:**
- OWASP ZAP scan
- JWT token validation tests
- Brute force protection tests
- CORS policy tests

**Definition of Done:**
- Apache Shiro removed
- Spring Security 6+ implemented
- JWT authentication working
- Refresh token mechanism working
- RBAC implemented
- MFA implemented
- All authentication tests passing
- Security scan shows zero critical vulnerabilities
- Documentation complete
- Code review approved

---

### Story 5: API Security Implementation

**Story ID:** MOD-105  
**Story Points:** 8  
**Priority:** Critical  
**Assignee:** Backend Developer  
**Quarter:** Q1  
**Dependencies:** MOD-104

**User Story:**
As a backend developer, I need to implement proper CORS, rate limiting, and input validation so that APIs are protected from common attacks and abuse.

**Acceptance Criteria:**
- [ ] Configure CORS with whitelist of allowed origins
- [ ] Implement rate limiting per user and per endpoint
- [ ] Implement request validation with Bean Validation
- [ ] Implement SQL injection prevention
- [ ] Implement XSS prevention
- [ ] Add security headers (CSP, X-Frame-Options, HSTS)
- [ ] Implement request size limits
- [ ] Add API versioning
- [ ] Test all security measures
- [ ] Document API security guidelines

**Technical Details:**

**Rate Limiting (Bucket4j):**
```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.7.0</version>
</dependency>
```

**RateLimitingFilter.java:**
```java
@Component
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        
        String key = getClientKey(request);
        Bucket bucket = resolveBucket(key);
        
        if (bucket.tryConsume(1)) {
            filterChain.doFilter(request, response);
        } else {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Too many requests");
        }
    }
    
    private Bucket resolveBucket(String key) {
        return cache.computeIfAbsent(key, k -> createNewBucket());
    }
    
    private Bucket createNewBucket() {
        Bandwidth limit = Bandwidth.builder()
                .capacity(100)
                .refillGreedy(100, Duration.ofMinutes(1))
                .build();
        return Bucket.builder()
                .addLimit(limit)
                .build();
    }
    
    private String getClientKey(HttpServletRequest request) {
        String userId = extractUserId(request);
        return userId != null ? userId : request.getRemoteAddr();
    }
}
```

**Input Validation:**
```java
@RestController
@RequestMapping("/api/v1/citizens")
@Validated
public class CitizenController {
    
    @PostMapping
    public ResponseEntity<CitizenResponse> createCitizen(
            @Valid @RequestBody CreateCitizenRequest request
    ) {
        // Request is automatically validated
        return ResponseEntity.ok(citizenService.create(request));
    }
}

@Data
public class CreateCitizenRequest {
    
    @NotBlank(message = "Voter ID is required")
    @Pattern(regexp = "^[A-Z]{3}[0-9]{7}$", message = "Invalid voter ID format")
    private String voterId;
    
    @NotBlank(message = "First name is required")
    @Size(min = 2, max = 100, message = "First name must be between 2 and 100 characters")
    private String firstName;
    
    @NotBlank(message = "Family name is required")
    @Size(min = 2, max = 100, message = "Family name must be between 2 and 100 characters")
    private String familyName;
    
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120, message = "Age must be at most 120")
    private Integer age;
    
    @Pattern(regexp = "^[0-9]{10}$", message = "Invalid mobile number")
    private String mobile;
    
    @Email(message = "Invalid email address")
    private String email;
}
```

**SQL Injection Prevention:**
```java
// Use JPA/Hibernate with parameterized queries
@Repository
public interface CitizenRepository extends JpaRepository<Citizen, Long> {
    
    // Safe - uses parameterized query
    @Query("SELECT c FROM Citizen c WHERE c.voterId = :voterId")
    Optional<Citizen> findByVoterId(@Param("voterId") String voterId);
    
    // Safe - Spring Data method
    List<Citizen> findByStateAndAssemblyNo(String state, String assemblyNo);
}
```

**XSS Prevention:**
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new XSSInterceptor());
    }
}

public class XSSInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler
    ) throws Exception {
        // Sanitize request parameters
        Map<String, String[]> parameterMap = request.getParameterMap();
        for (Map.Entry<String, String[]> entry : parameterMap.entrySet()) {
            String[] values = entry.getValue();
            for (int i = 0; i < values.length; i++) {
                values[i] = sanitize(values[i]);
            }
        }
        return true;
    }
    
    private String sanitize(String value) {
        return value.replaceAll("<", "&lt;")
                    .replaceAll(">", "&gt;")
                    .replaceAll("\"", "&quot;")
                    .replaceAll("'", "&#x27;")
                    .replaceAll("/", "&#x2F;");
    }
}
```

**Security Headers:**
```java
@Configuration
public class SecurityHeadersConfig {
    
    @Bean
    public FilterRegistrationBean<SecurityHeadersFilter> securityHeadersFilter() {
        FilterRegistrationBean<SecurityHeadersFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new SecurityHeadersFilter());
        registrationBean.addUrlPatterns("/api/*");
        return registrationBean;
    }
}

public class SecurityHeadersFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        
        // Content Security Policy
        response.setHeader("Content-Security-Policy", 
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';");
        
        // X-Frame-Options
        response.setHeader("X-Frame-Options", "DENY");
        
        // X-Content-Type-Options
        response.setHeader("X-Content-Type-Options", "nosniff");
        
        // X-XSS-Protection
        response.setHeader("X-XSS-Protection", "1; mode=block");
        
        // Strict-Transport-Security
        response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
        
        filterChain.doFilter(request, response);
    }
}
```

**Testing:**
```bash
# Test rate limiting
for i in {1..150}; do
  curl -X GET http://localhost:8080/api/v1/citizens
done
# Should get 429 after 100 requests

# Test input validation
curl -X POST http://localhost:8080/api/v1/citizens \
  -H "Content-Type: application/json" \
  -d '{"voterId":"INVALID","firstName":"A"}'
# Should get 400 with validation errors

# Test security headers
curl -I http://localhost:8080/api/v1/citizens
# Should see security headers in response
```

**Definition of Done:**
- CORS configured
- Rate limiting implemented
- Input validation working
- SQL injection prevention verified
- XSS prevention verified
- Security headers added
- All security tests passing
- Documentation complete
- Code review approved

---

### Story 6: Dependency Vulnerability Remediation

**Story ID:** MOD-106  
**Story Points:** 5  
**Priority:** High  
**Assignee:** Backend Developer + Mobile Developer  
**Quarter:** Q1

**User Story:**
As a developer, I need to update all dependencies to latest secure versions and implement Snyk scanning so that all known vulnerabilities are eliminated and future vulnerabilities are detected early.

**Acceptance Criteria:**
- [ ] Run dependency audit on backend (Maven)
- [ ] Run dependency audit on mobile (npm)
- [ ] Update all vulnerable dependencies
- [ ] Set up Snyk integration in GitHub
- [ ] Configure Snyk to fail builds on critical vulnerabilities
- [ ] Run security scans and fix all critical/high issues
- [ ] Document dependency update process
- [ ] Create dependency update policy

**Technical Details:**

**Backend (Maven):**
```bash
# Audit dependencies
mvn dependency-check:check

# Update dependencies
mvn versions:display-dependency-updates
mvn versions:use-latest-versions

# Generate security report
mvn dependency-check:aggregate
```

**Mobile (npm):**
```bash
# Audit dependencies
npm audit

# Fix vulnerabilities
npm audit fix

# Force fix (may have breaking changes)
npm audit fix --force

# Check for outdated packages
npm outdated

# Update packages
npm update
```

**Snyk Integration (.github/workflows/security-scan.yml):**
```yaml
name: Security Scan

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]
  schedule:
    - cron: '0 0 * * 0'  # Weekly scan

jobs:
  snyk-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/maven@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --fail-on=all
      
      - name: Upload result to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: snyk.sarif

  snyk-mobile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./IONICAPP
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN}}
        with:
          args: --severity-threshold=high --fail-on=all
          command: test
```

**Dependency Update Policy (DEPENDENCY_POLICY.md):**
```markdown
# Dependency Update Policy

## Update Frequency
- **Critical Security Updates**: Immediate (within 24 hours)
- **High Security Updates**: Within 1 week
- **Medium Security Updates**: Within 1 month
- **Low Security Updates**: Next quarterly release
- **Feature Updates**: Quarterly review

## Update Process
1. Review release notes and breaking changes
2. Update dependency version
3. Run full test suite
4. Perform security scan
5. Test in staging environment
6. Create PR with detailed change notes
7. Get approval from tech lead
8. Merge and deploy

## Monitoring
- Weekly automated Snyk scans
- Monthly dependency review meeting
- Quarterly major version upgrade review
```

**Testing:**
```bash
# Run security scans
mvn dependency-check:check
npm audit

# Verify no critical vulnerabilities
# Backend: Check target/dependency-check-report.html
# Mobile: Check npm audit output

# Test application after updates
mvn test
npm test
```

**Definition of Done:**
- All critical and high vulnerabilities fixed
- Snyk integrated in CI/CD
- Security scans passing
- Dependency update policy documented
- Team trained on security scanning
- Code review approved

---

## Priority 2: Testing Framework Implementation

### Story 7: Backend Unit Testing Setup

**Story ID:** MOD-107  
**Story Points:** 8  
**Priority:** High  
**Assignee:** Backend Developer + QA Engineer  
**Quarter:** Q1

**User Story:**
As a backend developer, I need to implement JUnit 5, Mockito, and Testcontainers for comprehensive unit testing so that code quality is maintained and regressions are prevented.

**Acceptance Criteria:**
- [ ] Set up JUnit 5 testing framework
- [ ] Set up Mockito for mocking
- [ ] Set up Testcontainers for integration tests
- [ ] Create test base classes and utilities
- [ ] Write unit tests for all service classes (target: 80% coverage)
- [ ] Write integration tests for repositories
- [ ] Write integration tests for controllers
- [ ] Configure code coverage with JaCoCo
- [ ] Set up coverage gates in CI/CD
- [ ] Document testing guidelines

**Technical Details:**

**pom.xml:**
```xml
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>postgresql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- Spring Boot Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- JaCoCo -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
                <execution>
                    <id>jacoco-check</id>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>PACKAGE</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**Unit Test Example (CitizenServiceTest.java):**
```java
@ExtendWith(MockitoExtension.class)
class CitizenServiceTest {
    
    @Mock
    private CitizenRepository citizenRepository;
    
    @Mock
    private CitizenMapper citizenMapper;
    
    @InjectMocks
    private CitizenService citizenService;
    
    @Test
    @DisplayName("Should create citizen successfully")
    void shouldCreateCitizenSuccessfully() {
        // Given
        CreateCitizenRequest request = new CreateCitizenRequest();
        request.setVoterId("ABC1234567");
        request.setFirstName("John");
        request.setFamilyName("Doe");
        
        Citizen citizen = new Citizen();
        citizen.setId(1L);
        citizen.setVoterId("ABC1234567");
        
        when(citizenMapper.toEntity(request)).thenReturn(citizen);
        when(citizenRepository.save(any(Citizen.class))).thenReturn(citizen);
        when(citizenMapper.toResponse(citizen)).thenReturn(new CitizenResponse());
        
        // When
        CitizenResponse response = citizenService.create(request);
        
        // Then
        assertThat(response).isNotNull();
        verify(citizenRepository).save(any(Citizen.class));
    }
    
    @Test
    @DisplayName("Should throw exception when voter ID already exists")
    void shouldThrowExceptionWhenVoterIdExists() {
        // Given
        CreateCitizenRequest request = new CreateCitizenRequest();
        request.setVoterId("ABC1234567");
        
        when(citizenRepository.existsByVoterId("ABC1234567")).thenReturn(true);
        
        // When & Then
        assertThatThrownBy(() -> citizenService.create(request))
                .isInstanceOf(DuplicateVoterIdException.class)
                .hasMessage("Voter ID already exists: ABC1234567");
    }
}
```

**Integration Test Example (CitizenRepositoryTest.java):**
```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class CitizenRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private CitizenRepository citizenRepository;
    
    @Test
    @DisplayName("Should find citizen by voter ID")
    void shouldFindCitizenByVoterId() {
        // Given
        Citizen citizen = new Citizen();
        citizen.setVoterId("ABC1234567");
        citizen.setFirstName("John");
        citizen.setFamilyName("Doe");
        citizenRepository.save(citizen);
        
        // When
        Optional<Citizen> found = citizenRepository.findByVoterId("ABC1234567");
        
        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getVoterId()).isEqualTo("ABC1234567");
    }
}
```

**Controller Test Example (CitizenControllerTest.java):**
```java
@WebMvcTest(CitizenController.class)
@Import(SecurityConfig.class)
class CitizenControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private CitizenService citizenService;
    
    @MockBean
    private JwtService jwtService;
    
    @Test
    @DisplayName("Should create citizen and return 201")
    @WithMockUser(roles = "ADMIN")
    void shouldCreateCitizenAndReturn201() throws Exception {
        // Given
        CreateCitizenRequest request = new CreateCitizenRequest();
        request.setVoterId("ABC1234567");
        request.setFirstName("John");
        request.setFamilyName("Doe");
        
        CitizenResponse response = new CitizenResponse();
        response.setId(1L);
        response.setVoterId("ABC1234567");
        
        when(citizenService.create(any(CreateCitizenRequest.class))).thenReturn(response);
        
        // When & Then
        mockMvc.perform(post("/api/v1/citizens")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.voterId").value("ABC1234567"));
    }
    
    @Test
    @DisplayName("Should return 400 when request is invalid")
    @WithMockUser(roles = "ADMIN")
    void shouldReturn400WhenRequestIsInvalid() throws Exception {
        // Given
        CreateCitizenRequest request = new CreateCitizenRequest();
        // Missing required fields
        
        // When & Then
        mockMvc.perform(post("/api/v1/citizens")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest());
    }
}
```

**Run Tests:**
```bash
# Run all tests
mvn test

# Run specific test class
mvn test -Dtest=CitizenServiceTest

# Run tests with coverage
mvn clean test jacoco:report

# View coverage report
open target/site/jacoco/index.html
```

**Definition of Done:**
- JUnit 5, Mockito, Testcontainers set up
- Unit tests written for all services (80%+ coverage)
- Integration tests written for repositories
- Integration tests written for controllers
- JaCoCo configured with 80% coverage gate
- All tests passing
- Testing guidelines documented
- Code review approved

---

Due to the length of this document, I'll continue with the remaining stories. This is story 7 of 48. Would you like me to continue with the remaining 41 stories across Q1-Q4?

**Remaining Stories Overview:**
- Q1: Stories 8-12 (Testing frameworks)
- Q2: Stories 13-22 (Architecture modernization & database migration)
- Q3: Stories 23-34 (Advanced services & infrastructure)
- Q4: Stories 35-48 (Mobile modernization & final integration)

Each story follows the same detailed format with acceptance criteria, technical details, code examples, testing procedures, and definition of done.

Would you like me to:
1. Continue with all remaining stories in this file?
2. Create separate files for each quarter?
3. Focus on specific quarters/stories?

