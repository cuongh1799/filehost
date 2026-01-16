# Applicant Profile Service - Complete Documentation

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Workflow](#workflow)
4. [Security System](#security-system)
5. [Routes & API Endpoints](#routes--api-endpoints)
6. [Code Structure](#code-structure)
7. [How to Modify Security](#how-to-modify-security)
8. [How to Modify Routes](#how-to-modify-routes)
9. [Setup & Running](#setup--running)
10. [Database Schema](#database-schema)
11. [Key Components](#key-components)

---

## Project Overview

**Applicant Profile** is a Java Spring Boot microservice that manages applicant profiles, including their education, work experience, job preferences, and skills. It's part of a larger ecosystem with API gateway integration and Kafka event-driven communication.

- **Technology Stack**: Java 21, Spring Boot 3.5.8, PostgreSQL, Kafka, Spring Security
- **Build Tool**: Maven
- **Port**: 8092
- **Service Name**: `applicant-profile`

---

## Architecture

### Module Structure

```
applicant_profile/
├── applicant_profile_api/      # API contracts & DTOs (interfaces/data transfer objects)
│   └── external/
│   │   └── ApplicantExternalService.java   (Kafka-based service for other services)
│   └── internal/
│       └── ApplicantInternalService.java   (REST API service for direct clients)
│
└── applicant_profile_service/  # Implementation & business logic
    ├── applicant/              # Core applicant entity & controller
    ├── education/              # Education record management
    ├── work_experience/        # Work experience management
    ├── job_preference/         # Job preference management
    ├── applicant_skill/        # Skills management
    ├── integration/            # External service integrations
    │   ├── tag/
    │   └── media/
    ├── common/
    │   ├── config/             # Security, OpenAPI, RequestFilter
    │   ├── http/               # Exception handling, response utilities
    │   ├── kafka/              # Kafka producer & consumer
    │   └── util/               # JWT handling, utilities
    └── resources/
        ├── application.properties
        └── db/migration/       # Flyway database migrations
```

### Two-Tier Architecture

1. **API Module** (`applicant_profile_api`)
   - Defines contracts (interfaces)
   - Contains shared DTOs
   - Prevents circular dependencies

2. **Service Module** (`applicant_profile_service`)
   - Implements business logic
   - Handles database operations
   - Manages HTTP & Kafka communication

---

## Workflow

### 1. Request Flow (REST API)

```
Client Request
    ↓
API Gateway (path: /api/applicant-profile)
    ↓
RequestFilter (Security - JWT/JWE Token Validation)
    ↓
SecurityConfig (Authorization checks)
    ↓
ApplicantController (REST endpoint handler)
    ↓
ApplicantInternalService (Business logic)
    ↓
Database (PostgreSQL via JPA)
    ↓
Response
```

### 2. Kafka Event Flow (Inter-Service Communication)

```
External Service Request
    ↓
Kafka Topic (e.g., "applicant.create.req")
    ↓
EventConsumerImpl (Kafka listener)
    ↓
ApplicantExternalService (Business logic)
    ↓
Database Operations
    ↓
Kafka Response Topic (e.g., "applicant.create.res")
    ↓
External Service Response
```

### 3. Authentication Flow

```
Client sends Request with "Authorization: Bearer <JWE_TOKEN>"
    ↓
RequestFilter intercepts request
    ↓
Extract token from header
    ↓
Decrypt JWE to get JWS
    ↓
Parse & validate JWS (signature, expiration)
    ↓
Extract: id, email, role
    ↓
Set Spring Security Context
    ↓
Request proceeds to controller
```

---

## Security System

### Overview

The security model uses **JWT** with **dual-layer encryption**:
- **JWE (JSON Web Encryption)**: Outer encryption for confidentiality
- **JWS (JSON Web Signature)**: Inner signature for integrity & authenticity

### How It Works

#### 1. Token Structure
```
┌─────────────────────────────┐
│  JWE Token (Encrypted)      │
│  ┌───────────────────────┐  │
│  │ JWS Token (Signed)    │  │
│  │ ┌─────────────────┐   │  │
│  │ │ Claims          │   │  │
│  │ │ - id (UUID)     │   │  │
│  │ │ - email         │   │  │
│  │ │ - role          │   │  │
│  │ └─────────────────┘   │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

#### 2. Request Processing
```java
// In RequestFilter.java
String authHeader = request.getHeader("Authorization");
// Expected format: "Bearer <JWE_TOKEN>"

String jweToken = authHeader.substring(7);  // Remove "Bearer "
String jwsToken = jwtService.decryptJweToken(jweToken);  // Decrypt JWE
ParseTokenDto tokenData = jwtService.parseJwsToken(jwsToken);  // Validate JWS

// Extract claims
String id = tokenData.getId();
String email = tokenData.getEmail();
String role = tokenData.getRole();

// Set Spring Security context
UsernamePasswordAuthenticationToken authentication = 
    new UsernamePasswordAuthenticationToken(
        email,
        null,
        Collections.singletonList(
            new SimpleGrantedAuthority("ROLE_" + role)
        )
    );
SecurityContextHolder.getContext().setAuthentication(authentication);
```

#### 3. Authorization Rules (in SecurityConfig.java)

```
┌─────────────────────────────────────────────────────────────┐
│                   Authorization Rules                        │
├─────────────────────────────────────────────────────────────┤
│ GET  /profile/**                  → PERMIT_ALL (public)     │
│ POST /profile/**                  → AUTHENTICATED required  │
│ PUT  /profile/**                  → AUTHENTICATED required  │
│ DELETE /profile/**                → AUTHENTICATED required  │
│                                                              │
│ GET /swagger-ui/**, /v3/api-docs → PERMIT_ALL (public)     │
│ GET /actuator/**                  → PERMIT_ALL (public)     │
│                                                              │
│ Any other request                 → AUTHENTICATED required  │
└─────────────────────────────────────────────────────────────┘
```

#### 4. Keystore Configuration
```properties
# application.properties
key-store.type=PKCS12
key-store.path=classpath:${KEYSTORE_FILENAME}
key-store.password=${KEYSTORE_PWD}
key-store.alias.jws=${KEYSTORE_ALIAS_JWS}      # For signing
key-store.alias.jwe=${KEYSTORE_ALIAS_JWE}      # For encryption
```

### Session Management
```java
.sessionManagement(session -> 
    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
)
```
- **STATELESS**: No HTTP sessions. Each request is independent and must include authentication token.
- Suitable for microservices & APIs

### CSRF Protection
```java
.csrf(csrf -> csrf.disable())
```
- Disabled because: 
  - JWT tokens are used instead of session cookies
  - Tokens sent in `Authorization` header (not cookies)
  - Stateless design eliminates CSRF vulnerability

---

## Routes & API Endpoints

### Base Paths

```
Direct Service:    http://localhost:8092
Via Gateway:       http://localhost:8080/api/applicant-profile
Swagger UI:        http://localhost:8092/swagger-ui/index.html
OpenAPI JSON:      http://localhost:8092/v3/api-docs
```

### 1. Profile Management (Core)

#### Get Profile by ID
```http
GET /profile/{id}
Authorization: Bearer <token>  (Optional - GET is public)
Response: 200 OK
{
  "id": "uuid",
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "objective": "...",
  "phone": "...",
  "country": "...",
  "city": "...",
  "address": "..."
}
```

#### Get All Profiles (with filters)
```http
GET /profile?name=john&location=NYC&degree_type=BACHELOR&experience_type=FULL_TIME&experience_keywords=Java,Spring&employment_types=FULL_TIME
Authorization: Bearer <token>  (Optional)
Response: 200 OK
[
  { id, email, firstName, lastName, ... },
  ...
]
```

#### Create Profile
```http
POST /profile
Authorization: Bearer <token>  (Required)
Content-Type: application/json
{
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "objective": "...",
  "phone": "...",
  "country": "...",
  "city": "...",
  "address": "..."
}
Response: 201 CREATED
```

#### Update Profile
```http
PUT /profile/{id}
Authorization: Bearer <token>  (Required)
Content-Type: application/json
{ /* same body as POST */ }
Response: 200 OK
```

### 2. Education Management

#### Get Education
```http
GET /profile/{id}/education/{educationId}
Response: 200 OK
{
  "id": "uuid",
  "institution": "MIT",
  "degreeType": "BACHELOR",
  "gpa": 3.8,
  "startedAt": "2020-01-15",
  "endedAt": "2024-05-30"
}
```

#### Add Education
```http
POST /profile/{id}/education
Authorization: Bearer <token>  (Required)
{
  "institution": "MIT",
  "degreeType": "BACHELOR",
  "gpa": 3.8,
  "startedAt": "2020-01-15",
  "endedAt": "2024-05-30"
}
Response: 201 CREATED
```

#### Update Education
```http
PUT /profile/{id}/education/{educationId}
Authorization: Bearer <token>  (Required)
{ /* same body as POST */ }
Response: 200 OK
```

#### Delete Education
```http
DELETE /profile/{id}/education/{educationId}
Authorization: Bearer <token>  (Required)
Response: 204 NO CONTENT
```

### 3. Work Experience Management

#### Get Work Experience
```http
GET /profile/{id}/work-experience/{workExperienceId}
Response: 200 OK
{
  "id": "uuid",
  "title": "Senior Java Developer",
  "description": "...",
  "startedAt": "2022-01-15",
  "endedAt": "2024-05-30"
}
```

#### Add Work Experience
```http
POST /profile/{id}/work-experience
Authorization: Bearer <token>  (Required)
{
  "title": "Senior Java Developer",
  "description": "...",
  "startedAt": "2022-01-15",
  "endedAt": "2024-05-30"
}
Response: 201 CREATED
```

#### Update Work Experience
```http
PUT /profile/{id}/work-experience/{workExperienceId}
Authorization: Bearer <token>  (Required)
{ /* same body as POST */ }
Response: 200 OK
```

#### Delete Work Experience
```http
DELETE /profile/{id}/work-experience/{workExperienceId}
Authorization: Bearer <token>  (Required)
Response: 204 NO CONTENT
```

### 4. Job Preference Management

#### Get Job Preference
```http
GET /profile/{id}/job-preference/{jobPreferenceId}
Response: 200 OK
{
  "id": "uuid",
  "country": "USA",
  "isFresher": false,
  "employmentTypes": [123],  // Bitmask for employment types
  "salaryMin": 100000,
  "salaryMax": 150000
}
```

#### Add Job Preference
```http
POST /profile/{id}/job-preference
Authorization: Bearer <token>  (Required)
{
  "country": "USA",
  "isFresher": false,
  "employmentTypes": [123],
  "salaryMin": 100000,
  "salaryMax": 150000
}
Response: 201 CREATED
```

#### Update Job Preference
```http
PUT /profile/{id}/job-preference/{jobPreferenceId}
Authorization: Bearer <token>  (Required)
{ /* same body as POST */ }
Response: 200 OK
```

#### Delete Job Preference
```http
DELETE /profile/{id}/job-preference/{jobPreferenceId}
Authorization: Bearer <token>  (Required)
Response: 204 NO CONTENT
```

### 5. Skills Management

#### Get Skills
```http
GET /profile/{id}/skills
Response: 200 OK
[
  { "id": "uuid", "name": "Java", ... },
  { "id": "uuid", "name": "Spring", ... }
]
```

#### Update Skills
```http
PUT /profile/{id}/skills
Authorization: Bearer <token>  (Required)
Content-Type: application/json
["uuid-skill-1", "uuid-skill-2", "uuid-skill-3"]
Response: 200 OK
```

### 6. Media Management

#### Add/Update Media
```http
POST /profile/{id}/media
Authorization: Bearer <token>  (Required)
{
  "mediaType": "RESUME",
  "mediaUrl": "s3://bucket/resume.pdf",
  "mediaMetadata": { ... }
}
Response: 201 CREATED
{
  "id": "uuid",
  "mediaType": "RESUME",
  "mediaUrl": "s3://bucket/resume.pdf"
}
```

### 7. Search Endpoints

#### Full-Text Search
```http
GET /search/full-text?query=java&location=NYC&degree_type=BACHELOR&experience_type=FULL_TIME&experience_keywords=Java,Spring&skill_ids=uuid1,uuid2&page=0&size=10
Response: 200 OK
{
  "content": [ /* profiles */ ],
  "totalElements": 42,
  "totalPages": 5,
  "currentPage": 0,
  "pageSize": 10
}
```

### 8. Health Check
```http
GET /greeting
Response: 200 OK
"greeting from service applicant profile service"
```

---

## Code Structure

### Key Directories

#### `/applicant`
**Main profile management**
- `ApplicantController.java` - REST endpoints
- `ApplicantInternalService.java` - Business logic interface
- `ApplicantInternalServiceImpl.java` - Implementation
- `ApplicantModel.java` - JPA entity
- `ApplicantRepository.java` - Database access

#### `/education`
**Education records**
- `EducationModel.java` - JPA entity
- `EducationRepository.java` - Database access
- `EducationService.java` - Business logic

#### `/work_experience`
**Work history**
- `WorkExperienceModel.java` - JPA entity
- `WorkExperienceRepository.java` - Database access
- `WorkExperienceService.java` - Business logic

#### `/job_preference`
**Job preferences**
- `JobPreferenceModel.java` - JPA entity
- `JobPreferenceRepository.java` - Database access
- `JobPreferenceService.java` - Business logic

#### `/applicant_skill`
**Skills linking**
- `ApplicantSkillModel.java` - JPA entity
- `ApplicantSkillRepository.java` - Database access

#### `/common/config`
**Framework & security configuration**
- `SecurityConfig.java` - Spring Security setup
- `RequestFilter.java` - JWT token validation
- `OpenApiConfig.java` - Swagger/OpenAPI setup
- `OpenAPIDefinition` - API documentation

#### `/common/http`
**HTTP utilities & error handling**
- `ExceptionController.java` - Global exception handler
- `GenericResponseDto.java` - Standard response wrapper
- `ExceptionDto.java` - Error response structure

#### `/common/kafka`
**Event messaging**
- `EventProducerImpl.java` - Sends Kafka events
- `EventConsumerImpl.java` - Receives Kafka events

#### `/common/util`
**Utilities**
- `JwtService.java` - JWT encryption/decryption & validation
- `ParseTokenDto.java` - Parsed token data
- `CreateTokenDto.java` - Token creation data

#### `/integration`
**External service integrations**
- `tag/` - Tag service integration
- `media/` - Media service integration

---

## How to Modify Security

### 1. Change Authorization Rules

**File**: [SecurityConfig.java](applicant_profile_service/src/main/java/devision/wukong/applicant_profile_service/common/config/SecurityConfig.java)

#### Example: Allow everyone to create profiles (currently requires authentication)

```java
// BEFORE (authenticated required for POST)
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.POST, "/profile/**")
    .authenticated()
    ...
)

// AFTER (public)
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.POST, "/profile").permitAll()
    .requestMatchers(HttpMethod.POST, "/profile/**")
    .authenticated()
    ...
)
```

#### Example: Require authentication for all GET requests

```java
// BEFORE
.requestMatchers(HttpMethod.GET).permitAll()

// AFTER
.requestMatchers(HttpMethod.GET).authenticated()
```

#### Example: Add role-based access control

```java
.authorizeHttpRequests(auth -> auth
    // Only ADMIN can delete profiles
    .requestMatchers(HttpMethod.DELETE, "/profile/**")
    .hasRole("ADMIN")
    // RECRUITER and ADMIN can create
    .requestMatchers(HttpMethod.POST, "/profile/**")
    .hasAnyRole("RECRUITER", "ADMIN")
    // Everyone can read
    .requestMatchers(HttpMethod.GET)
    .permitAll()
    .anyRequest().authenticated()
)
```

### 2. Enable Session-Based Authentication (instead of Stateless)

```java
// BEFORE (stateless)
.sessionManagement(session -> 
    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
)

// AFTER (stateful)
.sessionManagement(session -> 
    session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
)
```

### 3. Enable CSRF Protection (for session-based auth)

```java
// BEFORE
.csrf(csrf -> csrf.disable())

// AFTER
.csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
```

### 4. Modify JWT Token Handling

**File**: [RequestFilter.java](applicant_profile_service/src/main/java/devision/wukong/applicant_profile_service/common/config/RequestFilter.java)

#### Example: Accept tokens from custom header instead of Authorization

```java
// BEFORE
String authHeader = request.getHeader("Authorization");
if (authHeader != null && authHeader.startsWith("Bearer ")) {
    String jweToken = authHeader.substring(7);

// AFTER (use custom header "X-API-Token")
String authHeader = request.getHeader("X-API-Token");
if (authHeader != null && !authHeader.isEmpty()) {
    String jweToken = authHeader;
```

#### Example: Skip authentication for specific paths

```java
@Override
protected void doFilterInternal(...) {
    String path = request.getRequestURI();
    
    // Skip filter for public endpoints
    if (path.startsWith("/swagger-ui") || 
        path.startsWith("/v3/api-docs") ||
        path.startsWith("/actuator")) {
        filterChain.doFilter(request, response);
        return;
    }
    
    // ... rest of token validation
}
```

### 5. Add Custom Authentication Provider

```java
@Bean
public AuthenticationProvider customAuthProvider() {
    return new DaoAuthenticationProvider();
}

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authenticationProvider(customAuthProvider());
    // ...
}
```

### 6. Disable Security for Testing

```java
@Configuration
@Profile("test")
public class SecurityTestConfig {
    @Bean
    public SecurityFilterChain testSecurityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

---

## How to Modify Routes

### 1. Add a New Endpoint

**File**: [ApplicantController.java](applicant_profile_service/src/main/java/devision/wukong/applicant_profile_service/applicant/ApplicantController.java)

#### Example: Add endpoint to export profile as PDF

```java
@PostMapping("/profile/{id}/export/pdf")
public ResponseEntity<byte[]> exportProfileAsPdf(@PathVariable UUID id) {
    byte[] pdfBytes = applicantInternalService.generateProfilePdf(id);
    
    return ResponseEntity.ok()
        .header("Content-Disposition", "attachment; filename=profile.pdf")
        .contentType(MediaType.APPLICATION_PDF)
        .body(pdfBytes);
}
```

**Then implement in service**:
```java
// ApplicantInternalServiceImpl.java
public byte[] generateProfilePdf(UUID id) {
    GetProfileById profile = GetProfileById(id);
    // Generate PDF from profile data
    return pdfGenerator.generate(profile);
}
```

### 2. Change Endpoint Path

**Before**:
```java
@GetMapping("/profile/{id}")
public ResponseEntity<GetProfileById> getProfile(@PathVariable UUID id) { ... }
```

**After**:
```java
@GetMapping("/applicants/{id}")  // Changed from /profile/{id}
public ResponseEntity<GetProfileById> getProfile(@PathVariable UUID id) { ... }
```

**Update SecurityConfig**:
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.GET, "/applicants/**").permitAll()
    // ...
)
```

### 3. Change HTTP Method

**Before** (POST):
```java
@PostMapping("/profile/{id}/education")
public ResponseEntity<Void> addEducation(...) { ... }
```

**After** (PUT for idempotency):
```java
@PutMapping("/profile/{id}/education/{educationId}")
public ResponseEntity<Void> addOrUpdateEducation(...) { ... }
```

### 4. Add Query Parameters

**Before**:
```java
@GetMapping("/profile")
public ResponseEntity<List<FindApplicantsRes>> getProfileAll() { ... }
```

**After** (add salary filter):
```java
@GetMapping("/profile")
public ResponseEntity<List<FindApplicantsRes>> getProfileAll(
    @RequestParam(required = false) String name,
    @RequestParam(required = false) Integer salaryMin,
    @RequestParam(required = false) Integer salaryMax) {
    
    var result = applicantInternalService.getAllProfile(
        new FindApplicantsReq(name, salaryMin, salaryMax));
    return ResponseEntity.ok(result);
}
```

### 5. Add Path Variables

**Before**:
```java
@PutMapping("/profile")
public ResponseEntity<Void> updateProfile(...) { ... }
```

**After** (specify which field):
```java
@PutMapping("/profile/{id}/{fieldName}")
public ResponseEntity<Void> updateProfileField(
    @PathVariable UUID id,
    @PathVariable String fieldName,
    @RequestBody Object value) {
    
    applicantInternalService.updateProfileField(id, fieldName, value);
    return ResponseEntity.ok().build();
}
```

### 6. Change Request/Response Format

**Before**:
```java
@PostMapping("/profile")
public ResponseEntity<GetProfileById> createProfile(
    @RequestBody ProfileRequestDTO request) {
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(applicantInternalService.createProfile(request));
}
```

**After** (wrap in GenericResponseDto):
```java
@PostMapping("/profile")
public ResponseEntity<GenericResponseDto<GetProfileById>> createProfile(
    @RequestBody ProfileRequestDTO request) {
    GetProfileById result = applicantInternalService.createProfile(request);
    return GenericResponseDto.getGenericResponseDto(result);
}
```

### 7. Add Request Validation

**Before**:
```java
@PostMapping("/profile")
public ResponseEntity<GetProfileById> createProfile(
    @RequestBody ProfileRequestDTO request) { ... }
```

**After** (add validation):
```java
@PostMapping("/profile")
public ResponseEntity<GetProfileById> createProfile(
    @Valid @RequestBody ProfileRequestDTO request) { ... }
```

**Update DTO**:
```java
public record ProfileRequestDTO(
    @NotBlank(message = "Email is required") String email,
    @NotBlank(message = "First name is required") String firstName,
    @NotBlank(message = "Last name is required") String lastName,
    ...
) { }
```

### 8. Add Custom Controller for New Domain

**Create new controller file**: `ApplicantReferralController.java`

```java
@RestController
@RequestMapping("/referral")
public class ApplicantReferralController {
    
    @Autowired
    private ApplicantReferralService referralService;
    
    @GetMapping("/{id}")
    public ResponseEntity<ReferralDto> getReferral(@PathVariable UUID id) {
        return ResponseEntity.ok(referralService.getReferral(id));
    }
    
    @PostMapping("/{applicantId}")
    public ResponseEntity<ReferralDto> createReferral(
        @PathVariable UUID applicantId,
        @RequestBody CreateReferralReq request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(referralService.createReferral(applicantId, request));
    }
}
```

**Update SecurityConfig**:
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.POST, "/referral/**").authenticated()
    .requestMatchers(HttpMethod.GET, "/referral/**").permitAll()
    // ...
)
```

---

## Setup & Running

### Prerequisites

- Java 21
- Maven 3.6+
- PostgreSQL 12+
- Kafka (optional, for event-driven features)
- Docker (optional)

### Environment Variables

Create `.env` file in project root:

```properties
# Database
POSTGRES_URI=jdbc:postgresql://localhost:5432/applicant_profile
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Security - Keystore
KEYSTORE_FILENAME=keystore.p12
KEYSTORE_PWD=your-keystore-password
KEYSTORE_ALIAS_JWS=jws-key
KEYSTORE_ALIAS_JWE=jwe-key

# Eureka (Service Discovery)
EUREKA_SERVER_URI=http://localhost:8761/eureka

# Seed Data
SEED_KEY=your-seed-key
```

### Build

```bash
./mvnw clean install -DskipTests
```

### Run Locally

```bash
make run
# or
./mvnw spring-boot:run -pl applicant_profile_service
# or
java -jar applicant_profile_service/target/applicant_profile_service-0.0.1-SNAPSHOT.jar
```

### Run with Database Seeding

```bash
make run-seed
```

### Build Docker Image

```bash
docker build -t applicant-profile:latest --build-arg MY_MODULE=applicant_profile .
```

### Run Docker Container

```bash
docker run -p 8092:8092 \
  -e POSTGRES_URI=jdbc:postgresql://db:5432/applicant_profile \
  -e KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  applicant-profile:latest
```

---

## Database Schema

### Tables

#### `applicant`
```sql
id             UUID PRIMARY KEY
email          VARCHAR(255) UNIQUE NOT NULL
first_name     VARCHAR(255) NOT NULL
last_name      VARCHAR(255) NOT NULL
objective      VARCHAR(255) NOT NULL
phone          VARCHAR(255) UNIQUE NOT NULL
country        VARCHAR(255) NOT NULL
city           VARCHAR(255) NOT NULL
address        VARCHAR(255) NOT NULL
```

#### `education`
```sql
id             UUID PRIMARY KEY
applicant_id   UUID NOT NULL FOREIGN KEY -> applicant(id) CASCADE
institution    VARCHAR(255) NOT NULL
degree_type    VARCHAR(255) NOT NULL
gpa            DECIMAL NOT NULL
started_at     DATE NOT NULL
ended_at       DATE
```

#### `work_experience`
```sql
id             UUID PRIMARY KEY
applicant_id   UUID NOT NULL FOREIGN KEY -> applicant(id) CASCADE
title          VARCHAR(255) NOT NULL
description    TEXT
started_at     DATE NOT NULL
ended_at       DATE
```

#### `job_preference`
```sql
id             UUID PRIMARY KEY
applicant_id   UUID UNIQUE NOT NULL FOREIGN KEY -> applicant(id) CASCADE
country        VARCHAR(255) NOT NULL
is_fresher     BOOLEAN NOT NULL
employment_types INTEGER
salary_min     DECIMAL
salary_max     DECIMAL
```

#### `applicant_skill`
```sql
applicant_id   UUID NOT NULL FOREIGN KEY -> applicant(id) CASCADE
skill_id       UUID NOT NULL
PRIMARY KEY (applicant_id, skill_id)
```

### Database Migrations

Migrations are managed by **Flyway** in `/src/main/resources/db/migration/`:

- `V1__Initial_setup.sql` - Core tables
- `V2__insert_mock_data.sql` - Test data
- `V3__add_enum_types.sql` - Enumerations & constraints

To add a new migration:
1. Create file `V4__your_migration_name.sql`
2. Migrations run automatically on startup

---

## Key Components

### JpaRepository Pattern

All entities use Spring Data JPA for database operations:

```java
@Repository
public interface ApplicantRepository extends JpaRepository<ApplicantModel, UUID> {
    Optional<ApplicantModel> findByEmail(String email);
    List<ApplicantModel> findByCity(String city);
    // Custom query
    @Query("SELECT a FROM ApplicantModel a WHERE a.firstName LIKE %?1%")
    List<ApplicantModel> searchByFirstName(String name);
}
```

### Service Layer Pattern

Business logic is separated into service layer:

```java
@Service
public class ApplicantInternalServiceImpl implements ApplicantInternalService {
    @Autowired
    private ApplicantRepository applicantRepo;
    
    @Autowired
    private EducationService educationService;
    
    public GetProfileById GetProfileById(UUID id) {
        ApplicantModel applicant = applicantRepo.findById(id)
            .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND, "Profile not found"));
        
        // Map to DTO and aggregate related data
        return new GetProfileById(
            applicant.getId(),
            applicant.getEmail(),
            // ... other fields
        );
    }
}
```

### Kafka Event Handling

**Producer** (sending events):
```java
@Component
public class EventProducerImpl implements EventProducer {
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Override
    public void notifyApplicantProfileUpdate(NotifyApplicantUpdateReq req) {
        kafkaTemplate.send(ApplicantProfileTopicRegistry.UPDATE, req);
    }
}
```

**Consumer** (receiving events):
```java
@Component
public class EventConsumerImpl {
    @KafkaListener(topics = ApplicantProfileTopicRegistry.SAVE_REQ)
    @SendTo(ApplicantProfileTopicRegistry.SAVE_RES)
    public CreateApplicantRes createApplicantProfile(CreateApplicantReq req) {
        return applicantExternalService.createApplicantProfile(req);
    }
}
```

### Exception Handling

Global exception handler for consistent error responses:

```java
@ControllerAdvice
public class ExceptionController {
    @ExceptionHandler(ResponseStatusException.class)
    public ResponseEntity<?> handleNotFound(ResponseStatusException e) {
        return new ResponseEntity<>(
            new ExceptionDto(e.getReason(), e.getStatusCode().value()),
            e.getStatusCode()
        );
    }
}
```

---

## Common Tasks

### Add a New Entity & Endpoints

1. **Create Entity Model**
   ```java
   @Entity
   @Table(name = "certification")
   public class CertificationModel {
       @Id private UUID id;
       @ManyToOne
       @JoinColumn(name = "applicant_id")
       private ApplicantModel applicant;
       // fields
   }
   ```

2. **Create Repository**
   ```java
   @Repository
   public interface CertificationRepository 
       extends JpaRepository<CertificationModel, UUID> { }
   ```

3. **Create DTOs** in `applicant_profile_api`
4. **Create Service** with business logic
5. **Add Endpoints** to controller
6. **Update SecurityConfig** if needed
7. **Create Database Migration**

### Change JWT Token Duration

**File**: Configuration where JWT is created

Find the token creation logic and modify the `expiresIn` claim:

```java
// Before: 1 hour
claims.put("exp", System.currentTimeMillis() + 3600000);

// After: 7 days
claims.put("exp", System.currentTimeMillis() + (7 * 24 * 3600 * 1000));
```

### Enable Request Logging

Add to `application.properties`:

```properties
logging.level.org.springframework.security=DEBUG
logging.level.org.springframework.web=DEBUG
logging.level.devision.wukong=DEBUG
```

### Add Pagination Support

Create paginated response wrapper and use Spring's `Pageable`:

```java
@GetMapping("/profile")
public ResponseEntity<Page<FindApplicantsRes>> getProfileAll(
    @PageableDefault(size = 20) Pageable pageable) {
    
    Page<FindApplicantsRes> result = 
        applicantInternalService.getAllProfile(pageable);
    return ResponseEntity.ok(result);
}
```

---

## Troubleshooting

### 401 Unauthorized
- Check token validity & expiration
- Verify keystore path & password in `application.properties`
- Ensure `Authorization: Bearer <token>` header format

### 403 Forbidden
- User role doesn't match endpoint requirements
- Check SecurityConfig authorization rules

### Database Connection Error
- Verify PostgreSQL is running
- Check `POSTGRES_URI`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
- Ensure database exists

### Kafka Connection Error
- Verify Kafka broker is running
- Check `KAFKA_BOOTSTRAP_SERVERS` configuration

---

## Additional Resources

- [Spring Security Documentation](https://spring.io/projects/spring-security)
- [JWT Best Practices](https://tools.ietf.org/html/rfc7519)
- [Spring Data JPA](https://spring.io/projects/spring-data-jpa)
- [Kafka with Spring](https://spring.io/projects/spring-kafka)
- [Flyway Database Migration](https://flywaydb.org/)

