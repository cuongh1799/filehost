# Job Application Service

A microservice for managing job applications with JWT authentication, built with Spring Boot and following a multi-module Maven architecture.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
  - [Request Flow](#request-flow)
  - [Security Architecture](#security-architecture)
  - [Routing System](#routing-system)
- [Security Deep Dive](#security-deep-dive)
  - [JWT Token Structure](#jwt-token-structure)
  - [Authentication Flow](#authentication-flow)
  - [How to Change Security Rules](#how-to-change-security-rules)
- [API Endpoints](#api-endpoints)
- [Configuration](#configuration)
- [Development Guide](#development-guide)
  - [Setup](#setup)
  - [Building & Running](#building--running)
  - [Adding New Routes](#adding-new-routes)
  - [Modifying Security](#modifying-security)
- [Integration with Other Services](#integration-with-other-services)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

This is a **multi-module Maven project** with the following structure:

```
job_application/
├── job_application_api/      # Shared DTOs and interfaces
└── job_application_service/   # Main Spring Boot application
```

**Key Technologies:**
- **Spring Boot 3.5.8** - Main framework
- **Spring Security** - Authentication & Authorization
- **PostgreSQL** - Database
- **Kafka** - Event streaming for inter-service communication
- **JWT (JWS + JWE)** - Token-based authentication
- **Eureka** - Service discovery
- **OpenAPI/Swagger** - API documentation

---

## Project Structure

### job_application_api (Shared Module)

Contains interfaces and DTOs shared between services:

```
job_application_api/
└── src/main/java/devision/wukong/job_application_api/
    ├── external/
    │   ├── EventProducer.java               # Kafka event producer interface
    │   └── JobApplicationExternalService.java
    └── internal/
        ├── JobApplicationInternalService.java  # Service interface
        └── dto/                                 # Data Transfer Objects
            ├── GetJobApplicationDTO.java
            ├── JobApplicationDTO.java
            ├── JobApplicationStatus.java
            └── ListJobApplicationDto.java
```

### job_application_service (Main Application)

The main Spring Boot application:

```
job_application_service/
└── src/main/java/devision/wukong/job_application_service/
    ├── JobApplicationApplication.java       # Main application entry point
    ├── common/
    │   ├── config/
    │   │   ├── DataSeedConfig.java          # Database seeding
    │   │   ├── KafkaConfig.java             # Kafka configuration
    │   │   ├── OpenApiConfig.java           # Swagger/OpenAPI config
    │   │   ├── RequestFilter.java           # JWT authentication filter
    │   │   └── SecurityConfig.java          # Spring Security config
    │   ├── http/
    │   │   └── GenericResponseDto.java      # Standard API response wrapper
    │   ├── kafka/
    │   │   └── [Kafka consumers/producers]
    │   └── security/
    │       ├── JwtService.java              # JWT creation/parsing
    │       ├── JwtServiceImpl.java
    │       ├── KeyService.java              # Keystore management
    │       └── KeyServiceImpl.java
    └── job_application/
        ├── JobApplicationController.java    # REST API endpoints
        ├── JobApplicationInternalServiceImpl.java  # Business logic
        ├── JobApplicationExternalServiceImpl.java
        ├── JobApplicationModel.java         # JPA entity
        ├── JobApplicationRepo.java          # Database repository
        ├── JobApplicationSpecification.java # Query specifications
        └── MediaIntegrationService.java     # Media service integration
```

---

## How It Works

### Request Flow

```
1. Client sends HTTP request with Bearer token
   ↓
2. RequestFilter intercepts (before Spring Security)
   ↓
3. Extract JWT token from Authorization header
   ↓
4. Decrypt JWE token → get JWS token
   ↓
5. Parse and validate JWS token
   ↓
6. Set Spring SecurityContext with user authentication
   ↓
7. Spring Security checks authorization rules
   ↓
8. Request reaches Controller
   ↓
9. Controller calls Service layer
   ↓
10. Service interacts with Repository (database)
    ↓
11. Service may call other services via Kafka
    ↓
12. Response wrapped in GenericResponseDto
    ↓
13. JSON response sent to client
```

### Security Architecture

The security system has **three main components**:

#### 1. RequestFilter (JWT Authentication)

**Location:** `common/config/RequestFilter.java`

**Purpose:** Extracts and validates JWT tokens, sets Spring Security authentication context

**How it works:**
```java
1. Intercepts every incoming request
2. Checks for "Authorization: Bearer <token>" header
3. If token exists:
   a. Decrypt JWE (encrypted outer layer)
   b. Parse JWS (signed inner layer)
   c. Extract user info (id, email, role)
   d. Create Spring Security Authentication object
   e. Set SecurityContextHolder with authentication
4. If no token or invalid token:
   - Log warning
   - Clear security context
   - Request continues (SecurityConfig will decide if it's allowed)
```

**Key Code:**
```java
UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
    tokenData.getEmail(),
    null,
    Collections.singletonList(new SimpleGrantedAuthority("ROLE_" + tokenData.getRole()))
);
SecurityContextHolder.getContext().setAuthentication(authentication);
```

#### 2. SecurityConfig (Authorization Rules)

**Location:** `common/config/SecurityConfig.java`

**Purpose:** Defines which endpoints require authentication and which are public

**Current Rules:**
```java
- All GET requests → PUBLIC (no token required)
- All POST requests → AUTHENTICATED (token required)
- All PUT requests → AUTHENTICATED (token required)
- All DELETE requests → AUTHENTICATED (token required)
- Auth endpoints (/auth/*, /api/auth/*) → PUBLIC
- Swagger/docs → PUBLIC
- Actuator → PUBLIC
```

**How it works:**
- Spring Security evaluates rules **in order** from top to bottom
- First matching rule wins
- If SecurityContext has authentication → request is authenticated
- If no authentication and endpoint requires it → 403 Forbidden

#### 3. JWT Service (Token Management)

**Location:** `common/security/JwtServiceImpl.java`

**Purpose:** Creates and validates JWT tokens with double encryption

**Token Structure:**
```
JWE (Encrypted with RSA-OAEP-256 + AES-256-GCM)
  └── Contains: JWS (Signed with RS256)
        └── Contains: Payload {
              "sub": "user-id",
              "email": "user@example.com",
              "role": "APPLICANT",
              "iat": 1234567890,
              "exp": 1234654290
            }
```

**Why two layers?**
- **JWS (JSON Web Signature):** Ensures integrity - token hasn't been tampered with
- **JWE (JSON Web Encryption):** Ensures confidentiality - token content is encrypted

### Routing System

Routes are defined using **Spring MVC annotations** in the Controller:

```java
@RestController  // Marks class as REST controller
public class JobApplicationController {
    
    @GetMapping("/")           // GET http://localhost:8093/
    public ResponseEntity<...> getAll(...) { }
    
    @GetMapping("/{id}")       // GET http://localhost:8093/{id}
    public ResponseEntity<...> getById(@PathVariable UUID id) { }
    
    @PostMapping("/")          // POST http://localhost:8093/
    public ResponseEntity<...> create(@RequestBody ...) { }
    
    @PutMapping("/{id}")       // PUT http://localhost:8093/{id}
    public ResponseEntity<...> update(@PathVariable UUID id, ...) { }
}
```

**Important Notes:**
- `@PathVariable` - Extracts value from URL path (e.g., `/123` → `id=123`)
- `@RequestParam` - Extracts value from query string (e.g., `?status=PENDING`)
- `@RequestBody` - Parses JSON body into Java object
- No base path means endpoints start from root (`/`)

---

## Security Deep Dive

### JWT Token Structure

**Full Token Example:**
```
eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJlbmMiOiJBMjU2R0NNIn0.eyJzdWIiOiI1NTBlOGQw...
```

**Decryption Process:**

1. **Receive JWE token** (encrypted):
   ```
   eyJhbGciOiJSU0EtT0FFUC0yNTYi...
   ```

2. **Decrypt with RSA private key** → Get JWS token:
   ```
   eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
   ```

3. **Verify signature with RSA public key** → Get payload:
   ```json
   {
     "sub": "550e8d0e-e29b-41d4-a716-446655440000",
     "email": "john@example.com",
     "role": "APPLICANT",
     "iat": 1736693600,
     "exp": 1736780000
   }
   ```

### Authentication Flow

**For a Protected Endpoint (POST /)**

```
Client Request:
POST http://localhost:8093/
Authorization: Bearer eyJhbGci...
Content-Type: application/json
{
  "applicantId": "...",
  "jobPostId": "...",
  "status": "PENDING"
}

↓ RequestFilter processes token
↓ Sets SecurityContext with authentication
↓ Spring Security checks: POST requires authentication
✓ Authentication found in SecurityContext
✓ Request allowed

Controller executes:
  → Service layer processes business logic
  → Repository saves to database
  → Response returned

Response:
{
  "code": 200,
  "data": {
    "id": "...",
    "applicantId": "...",
    ...
  }
}
```

**For a Public Endpoint (GET /)**

```
Client Request:
GET http://localhost:8093/

↓ RequestFilter sees no token, does nothing
↓ Spring Security checks: GET is permitAll()
✓ Request allowed without authentication

Controller executes:
  → Service layer processes business logic
  → Repository queries database
  → Response returned
```

### How to Change Security Rules

#### Scenario 1: Make GET requests require authentication

**File:** `common/config/SecurityConfig.java`

**Change:**
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(/* public endpoints */).permitAll()
    // REMOVE: .requestMatchers(HttpMethod.GET, "/**").permitAll()
    // ADD:
    .requestMatchers(HttpMethod.GET, "/**").authenticated()
    .requestMatchers(HttpMethod.POST, "/**").authenticated()
    .requestMatchers(HttpMethod.PUT, "/**").authenticated()
    .requestMatchers(HttpMethod.DELETE, "/**").authenticated()
    .anyRequest().authenticated())
```

#### Scenario 2: Make specific endpoints public

**File:** `common/config/SecurityConfig.java`

**Add to permitAll() list:**
```java
.requestMatchers(
    // ... existing public endpoints ...
    "/job-applications/public",      // Make this endpoint public
    "/health",                        // Health check public
    "/status/**")                     // All status endpoints public
.permitAll()
```

#### Scenario 3: Require different roles for different endpoints

**Current system doesn't check roles, only authentication.**

**To add role-based access:**

1. **Modify SecurityConfig:**
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.POST, "/admin/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.POST, "/**").hasAnyRole("APPLICANT", "COMPANY")
    .requestMatchers(HttpMethod.GET, "/**").permitAll()
    .anyRequest().authenticated())
```

2. **RequestFilter already sets roles:**
```java
Collections.singletonList(new SimpleGrantedAuthority("ROLE_" + tokenData.getRole()))
```

#### Scenario 4: Disable security for development

**File:** `common/config/SecurityConfig.java`

**Replace entire securityFilterChain:**
```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
    return http.build();
}
```

---

## API Endpoints

### Base URL
```
http://localhost:8093
```

### Endpoints

| Method | Endpoint | Auth Required | Description |
|--------|----------|---------------|-------------|
| GET | `/` | ❌ No | List all job applications (with filters) |
| GET | `/{id}` | ❌ No | Get job application by ID |
| POST | `/` | ✅ Yes | Create new job application |
| PUT | `/{id}` | ✅ Yes | Update job application |
| GET | `/greeting` | ❌ No | Test endpoint |

### Query Parameters (GET /)

- `companyId` (UUID, optional) - Filter by company
- `applicantId` (UUID, optional) - Filter by applicant
- `jobPostUuid` (UUID, optional) - Filter by job post
- `status` (enum, optional) - Filter by status (PENDING, ACCEPTED, REJECTED, ARCHIVED)

### Request Examples

**Create Job Application (POST /)**
```bash
curl -X POST http://localhost:8093/ \
  -H "Authorization: Bearer eyJhbGci..." \
  -H "Content-Type: application/json" \
  -d '{
    "applicantId": "550e8d0e-e29b-41d4-a716-446655440000",
    "jobPostId": "089d3f5c-12d7-4ce4-8ce7-702582e97f53",
    "status": "PENDING",
    "description": "I am very interested in this position.",
    "createdAt": "2026-01-08T04:48:35.966Z",
    "mediaList": []
  }'
```

**Get All Job Applications (GET /)**
```bash
# No auth required
curl http://localhost:8093/

# With filters
curl "http://localhost:8093/?applicantId=550e8d0e-e29b-41d4-a716-446655440000&status=PENDING"
```

**Get Job Application by ID (GET /{id})**
```bash
# No auth required
curl http://localhost:8093/550e8400-e29b-41d4-a716-446655440000
```

**Update Job Application (PUT /{id})**
```bash
curl -X PUT http://localhost:8093/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer eyJhbGci..." \
  -H "Content-Type: application/json" \
  -d '{
    "status": "ACCEPTED"
  }'
```

---

## Configuration

### Environment Variables (.env file)

```bash
# Eureka Service Discovery
EUREKA_SERVER_URI=http://localhost:8761/eureka

# PostgreSQL Database
POSTGRES_URI=jdbc:postgresql://localhost:5454/ead_db?user=ead_user&password=ead_pwd

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# JWT Keystore (IMPORTANT!)
KEYSTORE_FILENAME=devision-wukong-keystore.p12    # Just the filename, not full path
KEYSTORE_PWD=devision-wukong-password
KEYSTORE_ALIAS_JWS=jws
KEYSTORE_ALIAS_JWE=jwe

# Optional: Data Seeding
SEED_KEY=your-secret-seed-key
```

### Application Properties

**File:** `job_application_service/src/main/resources/application.properties`

```properties
spring.application.name=job-application
server.port=8093

# Import environment variables
spring.config.import=optional:file:.env[.properties]

# Eureka
eureka.client.serviceUrl.defaultZone=${EUREKA_SERVER_URI}

# Database
spring.datasource.url=${POSTGRES_URI}
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Keystore
key-store.type=PKCS12
key-store.path=classpath:${KEYSTORE_FILENAME}
key-store.password=${KEYSTORE_PWD}
key-store.alias.jws=${KEYSTORE_ALIAS_JWS}
key-store.alias.jwe=${KEYSTORE_ALIAS_JWE}

# Kafka
spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP_SERVERS}
spring.kafka.consumer.group-id=${spring.application.name}-service
spring.kafka.consumer.auto-offset-reset=latest
spring.kafka.consumer.enable-auto-commit=false
```

### Keystore Setup

**CRITICAL:** The keystore file must be in `job_application_service/src/main/resources/`

```bash
# Place your keystore file
cp devision-wukong-keystore.p12 job_application_service/src/main/resources/

# Verify it exists
ls job_application_service/src/main/resources/devision-wukong-keystore.p12
```

**Common Error:**
```
Failed to load keystore from classpath:applicant_auth_service/src/main/resources/...
```

**Solution:** Your `KEYSTORE_FILENAME` should be JUST the filename, not a path:
```bash
# WRONG
KEYSTORE_FILENAME=applicant_auth_service/src/main/resources/devision-wukong-keystore.p12

# CORRECT
KEYSTORE_FILENAME=devision-wukong-keystore.p12
```

---

## Development Guide

### Setup

**Prerequisites:**
- Java 21
- Maven 3.9+
- PostgreSQL
- Kafka
- Eureka Server (running)

**Steps:**

1. **Clone and navigate to project:**
```bash
cd job_application/
```

2. **Configure environment variables:**
```bash
cp .env.example .env
nano .env  # Edit with your values
```

3. **Place keystore file:**
```bash
cp /path/to/devision-wukong-keystore.p12 job_application_service/src/main/resources/
```

4. **Build project:**
```bash
mvn clean install -DskipTests
```

5. **Run the service:**
```bash
make run
# OR
java -jar job_application_service/target/job_application_service-0.0.1-SNAPSHOT.jar
```

### Building & Running

**Using Makefile:**

```bash
# Build only
make build

# Build and run
make run

# Build and run with database seeding
make run-seed
```

**Using Maven directly:**

```bash
# Build
./mvnw clean package -DskipTests

# Run
java -jar job_application_service/target/job_application_service-0.0.1-SNAPSHOT.jar
```

**Using Docker:**

```bash
# Build Docker image
docker build -t job-application-service .

# Run container
docker run -p 8093:8093 --env-file .env job-application-service
```

### Adding New Routes

**Step 1: Define the endpoint in Controller**

**File:** `job_application/JobApplicationController.java`

```java
@RestController
public class JobApplicationController {
    
    // Add new endpoint
    @GetMapping("/status/{id}")
    public ResponseEntity<GenericResponseDto<String>> getStatus(@PathVariable UUID id) {
        String status = jobApplicationInternalService.getStatus(id);
        return GenericResponseDto.getGenericResponseDto(status);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<GenericResponseDto<Void>> delete(@PathVariable UUID id) {
        jobApplicationInternalService.delete(id);
        return GenericResponseDto.getGenericResponseDto(null);
    }
}
```

**Step 2: Add method to Service interface**

**File:** `job_application_api/internal/JobApplicationInternalService.java`

```java
public interface JobApplicationInternalService {
    // ... existing methods ...
    
    String getStatus(UUID id);
    void delete(UUID id);
}
```

**Step 3: Implement in Service**

**File:** `job_application/JobApplicationInternalServiceImpl.java`

```java
@Override
public String getStatus(UUID id) {
    JobApplicationModel model = jobApplicationRepo.findById(id)
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Not found"));
    return model.getStatus().toString();
}

@Override
public void delete(UUID id) {
    jobApplicationRepo.deleteById(id);
}
```

**Step 4: Configure security (if needed)**

**File:** `common/config/SecurityConfig.java`

```java
// By default:
// - GET /status/{id} → Public (no auth)
// - DELETE /{id} → Authenticated (requires token)

// To override:
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/status/**").authenticated() // Make it require auth
    .requestMatchers(HttpMethod.DELETE, "/**").hasRole("ADMIN")    // Admin only
    // ... rest of config
)
```

**Step 5: Test the endpoint**

```bash
# Public endpoint
curl http://localhost:8093/status/550e8400-e29b-41d4-a716-446655440000

# Protected endpoint
curl -X DELETE http://localhost:8093/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer eyJhbGci..."
```

### Modifying Security

#### Add Request Logging

**File:** `common/config/RequestFilter.java`

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {
    
    // Add logging
    log.info("Incoming request: {} {}", request.getMethod(), request.getRequestURI());
    
    String authHeader = request.getHeader("Authorization");
    // ... rest of filter
}
```

#### Change Token Expiration

**File:** `common/security/JwtServiceImpl.java`

```java
// Change from 1 day to 7 days
private final long jwsExpiration = 1000L * 60 * 60 * 24 * 7; // 7 days
```

#### Add IP Whitelisting

**File:** `common/config/RequestFilter.java`

```java
private static final Set<String> ALLOWED_IPS = Set.of("127.0.0.1", "192.168.1.100");

@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {
    
    String clientIP = request.getRemoteAddr();
    if (!ALLOWED_IPS.contains(clientIP)) {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.getWriter().write("IP not allowed");
        return;
    }
    
    // ... rest of filter
}
```

#### Add Custom Authentication Header

**File:** `common/config/RequestFilter.java`

```java
// Change from "Authorization" to "X-Auth-Token"
String authHeader = request.getHeader("X-Auth-Token");

if (authHeader != null && authHeader.startsWith("Bearer ")) {
    // ... rest of logic
}
```

---

## Integration with Other Services

This service communicates with other microservices via **Kafka events**.

### Outgoing Events (Published)

**File:** `job_application/JobApplicationExternalServiceImpl.java`

```java
// Implemented using EventProducer interface
eventProducer.listJobPostsByIds(request);    // Get job posts from job-post service
eventProducer.listApplicantsByIds(request);  // Get applicants from applicant service
eventProducer.listMedia(request);            // Get media from media service
```

### Incoming Events (Consumed)

**File:** `common/kafka/[Kafka Listeners]`

```java
// Listen for events from other services
@KafkaListener(topics = "job_post.deleted")
public void handleJobPostDeleted(JobPostDeletedEvent event) {
    // Archive or delete related job applications
}
```

### Event Flow Example

**Creating a Job Application:**

```
1. Client sends POST request with applicantId, jobPostId
   ↓
2. JobApplicationController receives request
   ↓
3. JobApplicationInternalServiceImpl.create()
   ↓
4. Validate applicant hasn't already applied
   ↓
5. Save job application to database
   ↓
6. Publish Kafka event: "media.save_req"
   ↓
7. Media service saves attachments
   ↓
8. Media service publishes: "media.save_res"
   ↓
9. This service consumes response
   ↓
10. Return complete job application with media URLs
```

---

## Troubleshooting

### Common Issues

#### 1. 403 Forbidden on POST/PUT/DELETE

**Symptom:**
```json
{
  "timestamp": "2026-01-12T12:00:00.000+00:00",
  "status": 403,
  "error": "Forbidden",
  "path": "/"
}
```

**Cause:** Token is not setting Spring Security authentication context.

**Solution:** Verify `RequestFilter` sets `SecurityContextHolder`:
```java
SecurityContextHolder.getContext().setAuthentication(authentication);
```

#### 2. Invalid Token: Tag mismatch

**Symptom:**
```
WARN RequestFilter : Invalid token: Cipher callback execution failed: Tag mismatch
```

**Causes:**
- Token was generated with different keystore
- Token is from auth service but this service uses different keys
- Token is corrupted

**Solution:**
- Ensure both services use the SAME keystore file
- Get a fresh token from auth service
- Verify keystore configuration is identical

#### 3. Keystore not found

**Symptom:**
```
Failed to load keystore from classpath:applicant_auth_service/src/main/resources/...
```

**Solution:**
```bash
# 1. Check .env file
cat .env | grep KEYSTORE_FILENAME
# Should be: KEYSTORE_FILENAME=devision-wukong-keystore.p12

# 2. Check file exists
ls job_application_service/src/main/resources/*.p12

# 3. Copy keystore if missing
cp /path/to/devision-wukong-keystore.p12 job_application_service/src/main/resources/

# 4. Rebuild
mvn clean install -DskipTests
```

#### 4. Invalid UUID string: jobapplication

**Symptom:**
```
Failed to convert value of type 'java.lang.String' to required type 'java.util.UUID'; 
Invalid UUID string: jobapplication
```

**Cause:** URL `/jobapplication` is matching `/{id}` route instead of being treated as its own route.

**Solution:** Add base path to controller:
```java
@RestController
@RequestMapping("/job-application")  // Add this
public class JobApplicationController {
    // Now routes become:
    // GET /job-application/
    // GET /job-application/{id}
}
```

#### 5. Port already in use

**Symptom:**
```
Web server failed to start. Port 8093 was already in use.
```

**Solution:**
```bash
# Find process using port
lsof -i :8093

# Kill process
kill -9 <PID>

# Or change port in application.properties
server.port=8094
```

#### 6. Cannot connect to Kafka

**Symptom:**
```
Error connecting to node localhost:9092
```

**Solution:**
```bash
# Check Kafka is running
docker ps | grep kafka

# Start Kafka
docker-compose up -d kafka

# Verify connectivity
telnet localhost 9092
```

---

## Development Workflow Summary

### Daily Development

```bash
# 1. Pull latest changes
git pull

# 2. Build project
mvn clean install -DskipTests

# 3. Run service
make run

# 4. Make changes to code
# Edit files in your IDE

# 5. Rebuild and restart
Ctrl+C  # Stop running service
mvn clean install -DskipTests
make run

# 6. Test changes
curl http://localhost:8093/
```

### Adding a New Feature

1. **Define API contract** in `job_application_api`
2. **Add endpoint** in `JobApplicationController`
3. **Implement business logic** in Service layer
4. **Update security** if needed in `SecurityConfig`
5. **Test endpoint** with curl/Postman
6. **Commit changes**

### Security Changes Checklist

- [ ] Update `SecurityConfig.java` with new rules
- [ ] Test with valid token (should work)
- [ ] Test without token (should get 403 if protected)
- [ ] Test with expired token (should get 401/403)
- [ ] Rebuild: `mvn clean install -DskipTests`
- [ ] Restart service
- [ ] Document changes in this README

---

## Additional Resources

- **Spring Security Docs:** https://docs.spring.io/spring-security/reference/
- **JWT.io:** https://jwt.io/ (Token decoder)
- **Spring Boot Docs:** https://docs.spring.io/spring-boot/docs/current/reference/html/
- **Kafka Docs:** https://kafka.apache.org/documentation/

---

## Notes

- Always rebuild after changing security configuration
- GET requests are public by default - be careful what data you expose
- Tokens are shared between services - use the same keystore
- Service runs on port **8093**
- Database updates automatically (`ddl-auto=update`)
- Kafka consumers run in background threads

---

**Last Updated:** January 12, 2026  
**Version:** 0.0.1-SNAPSHOT
