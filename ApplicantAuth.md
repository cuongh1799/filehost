# 🔐 Applicant Authentication Service

A microservice for applicant user authentication and authorization, featuring JWT-based security with JWE+JWS encryption, Google OAuth2 integration, and event-driven architecture using Kafka.

## 📑 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Security Architecture](#security-architecture)
- [How Authentication Works](#how-authentication-works)
- [Route Configuration](#route-configuration)
- [API Endpoints](#api-endpoints)
- [Event-Driven Architecture](#event-driven-architecture)
- [Configuration](#configuration)
- [Building & Running](#building--running)
- [Modifying Security](#modifying-security)
- [Modifying Routes](#modifying-routes)
- [Key Components Explained](#key-components-explained)

---

## 🏗️ Architecture Overview

This is a **Spring Boot microservice** with a modular architecture:

```
applicant_auth/
├── applicant_auth_api/        # Interface & DTOs (shared contracts)
│   └── src/main/java/
│       └── devision/wukong/applicant_auth_api/
│           ├── external/       # Kafka event producer interface
│           └── internal/       # Internal service interface & DTOs
│
├── applicant_auth_service/    # Implementation module
│   └── src/main/java/
│       └── devision/wukong/applicant_auth_service/
│           ├── ApplicantAuthApplication.java
│           ├── applicant_auth/    # Core auth logic
│           └── common/
│               ├── config/        # Spring configurations
│               ├── security/      # JWT & Key management
│               ├── oauth2/        # Google OAuth2
│               ├── kafka/         # Kafka producers
│               └── exception/     # Custom exceptions
```

### Design Pattern: **API-First with Separation of Concerns**
- **applicant_auth_api**: Defines contracts (interfaces and DTOs)
- **applicant_auth_service**: Implements business logic
- Other services can depend on `applicant_auth_api` without coupling to implementation

---

## 🛠️ Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Framework | Spring Boot 3.5.8 | Core application framework |
| Language | Java 21 | Programming language |
| Security | Spring Security | Authorization & authentication framework |
| Database | PostgreSQL | User credential storage |
| JWT Library | JJWT (io.jsonwebtoken) | Token creation & validation |
| Messaging | Apache Kafka | Event-driven communication |
| Service Discovery | Netflix Eureka | Microservice registration |
| API Documentation | Swagger/OpenAPI 3 | Interactive API docs |
| Build Tool | Maven | Dependency & build management |
| Containerization | Docker | Deployment packaging |

---

## 📂 Project Structure

### Core Modules

#### 1. **applicant_auth/** - Core Authentication Logic
```java
ApplicantAuthController.java         // REST endpoints
ApplicantAuthInternalServiceImpl.java // Business logic implementation
ApplicantAuthModel.java              // User entity (JPA)
ApplicantAuthRepo.java               // Data access layer
```

#### 2. **common/config/** - Application Configuration
```java
SecurityConfig.java        // Spring Security setup
RequestFilter.java         // JWT validation filter
KafkaProducerConfig.java   // Kafka producer setup
KafkaConsumerConfig.java   // Kafka consumer setup
OpenApiConfig.java         // Swagger documentation
DataSeedConfig.java        // Database seeding
```

#### 3. **common/security/** - Security Components
```java
JwtService.java          // JWT creation/parsing interface
JwtServiceImpl.java      // JWT implementation (JWS + JWE)
KeyService.java          // Key management interface
KeyServiceImpl.java      // KeyStore management
dto/
  ├── CreateTokenDto.java  // Token creation payload
  └── ParseTokenDto.java   // Parsed token data
```

#### 4. **common/oauth2/** - OAuth2 Integration
```java
GoogleOAuth2Service.java  // Google OAuth2 flow
```

#### 5. **common/kafka/** - Event Producers
```java
EventProducerImpl.java    // Kafka event publishing
```

---

## 🔒 Security Architecture

### Multi-Layer Security: JWE (Encryption) + JWS (Signature)

This service implements **nested JWT tokens** for maximum security:

```
┌─────────────────────────────────────────┐
│         JWE (Encrypted Container)       │
│  ┌───────────────────────────────────┐  │
│  │   JWS (Signed Token)              │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  Payload:                   │  │  │
│  │  │  - sub: userId              │  │  │
│  │  │  - email: user@example.com  │  │  │
│  │  │  - role: APPLICANT          │  │  │
│  │  │  - iat, exp                 │  │  │
│  │  └─────────────────────────────┘  │  │
│  │  Signature: RS256 (RSA)           │  │
│  └───────────────────────────────────┘  │
│  Encryption: RSA-OAEP-256 + A256GCM     │
└─────────────────────────────────────────┘
```

### Why This Approach?

1. **JWS (JSON Web Signature)** - RS256
   - Ensures token integrity and authenticity
   - Signed with RSA private key
   - Verified with RSA public key
   - Contains user claims (id, email, role)

2. **JWE (JSON Web Encryption)** - RSA-OAEP-256 + A256GCM
   - Encrypts the JWS token
   - Prevents token content inspection
   - Even if intercepted, attackers can't read claims

3. **RSA Key Pairs** stored in PKCS12 KeyStore:
   - `jws-keypair`: For signing/verifying tokens
   - `jwe-keypair`: For encrypting/decrypting tokens

### Security Flow

```mermaid
graph LR
    A[User Data] --> B[Create JWS Token]
    B --> C[Sign with JWS Private Key]
    C --> D[Encrypt with JWE Public Key]
    D --> E[Return JWE Token to Client]
    
    E --> F[Client sends JWE Token]
    F --> G[Decrypt with JWE Private Key]
    G --> H[Verify JWS with JWS Public Key]
    H --> I[Extract User Claims]
```

### KeyStore Management

**Location**: `src/main/resources/keystore.p12` (configured in `application.yml`)

**Structure**:
```
KeyStore (PKCS12)
├── Alias: jws
│   ├── Private Key: Signs tokens
│   └── Certificate: Contains public key for verification
└── Alias: jwe
    ├── Private Key: Decrypts tokens
    └── Certificate: Contains public key for encryption
```

**Key Loading** ([KeyServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/security/KeyServiceImpl.java)):
```java
private KeyStore getKeyStore() {
    keyStore = KeyStore.getInstance("PKCS12");
    Resource resource = resourceLoader.getResource(keystorePath);
    keyStore.load(inputStream, keystorePassword.toCharArray());
}
```

### Spring Security Configuration

**Configuration File**: [SecurityConfig.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/SecurityConfig.java)

**Key Features**:
- **CSRF**: Disabled (stateless API)
- **Session Management**: STATELESS (no server-side sessions)
- **Custom Filter**: `RequestFilter` validates JWT on every request

**Authorization Rules**:
```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.PUT, "/profile/**").authenticated()
    .requestMatchers("/api/auth/register", "/api/auth/login", ...).permitAll()
    .anyRequest().authenticated()
)
```

**Filter Chain**:
```
Request → RequestFilter → Spring Security → Controller
          (JWT validation)  (Authorization)
```

---

## 🔄 How Authentication Works

### 1. **Registration Flow**

```mermaid
sequenceDiagram
    participant Client
    participant Controller
    participant Service
    participant Database
    participant Kafka
    participant ProfileService
    participant EmailService

    Client->>Controller: POST /api/auth/register
    Controller->>Service: register(RegisterRequest)
    Service->>Database: Check if email exists
    Database-->>Service: Email available
    Service->>Database: Save new user (encrypted password)
    Database-->>Service: User saved with ID
    Service->>Kafka: Send CreateApplicantProfile event
    Kafka->>ProfileService: Create profile
    ProfileService-->>Kafka: Profile created (sync reply)
    Kafka-->>Service: Success
    Service->>Kafka: Send activation email event (async)
    Service->>Service: Generate JWE token
    Service-->>Client: Return token + user info
```

**Code Flow** ([ApplicantAuthInternalServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthInternalServiceImpl.java)):

```java
@Transactional
public ResponseEntity<AuthResponse> register(RegisterRequest request) {
    // 1. Validate email doesn't exist
    if (applicantAuthRepo.existsByEmail(request.getEmail())) {
        return ResponseEntity.status(BAD_REQUEST).body(...);
    }
    
    // 2. Create user with encrypted password
    ApplicantAuthModel savedUser = applicantAuthRepo.save(
        ApplicantAuthModel.builder()
            .id(UUID.randomUUID())
            .email(request.getEmail())
            .passwordHash(passwordEncoder.encode(request.getPassword()))
            .role("APPLICANT")
            .authType("LOCAL")
            .isVerified(false)
            .isActive(true)
            .build()
    );
    
    // 3. Create profile via Kafka (transaction rolls back on failure)
    eventProducer.createApplicantProfile(new CreateApplicantReq(...));
    
    // 4. Send activation email (fire-and-forget)
    eventProducer.sendAccountActivationEmail(new SendAccountActivationEmailReq(...));
    
    // 5. Generate JWT token
    String token = jwtService.createJweToken(
        new CreateTokenDto(savedUser.getId(), savedUser.getEmail(), savedUser.getRole())
    );
    
    return ResponseEntity.status(CREATED).body(
        AuthResponse.builder().token(token).build()
    );
}
```

**Transaction Safety**:
- If profile creation fails → entire transaction rolls back
- Email sending failure → logged but doesn't affect registration

---

### 2. **Login Flow**

```mermaid
sequenceDiagram
    participant Client
    participant Controller
    participant Service
    participant Database
    participant JwtService

    Client->>Controller: POST /api/auth/login {email, password}
    Controller->>Service: login(LoginRequest)
    Service->>Database: findByEmail(email)
    Database-->>Service: User found
    Service->>Service: Validate password (BCrypt)
    Service->>Service: Check isActive status
    Service->>JwtService: createJweToken(userId, email, role)
    JwtService->>JwtService: 1. Create JWS (signed)
    JwtService->>JwtService: 2. Encrypt JWS → JWE
    JwtService-->>Service: JWE token
    Service-->>Client: Return {token, user info}
```

**Code** ([ApplicantAuthInternalServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthInternalServiceImpl.java#L104-L151)):

```java
public ResponseEntity<AuthResponse> login(LoginRequest request) {
    // 1. Find user
    Optional<ApplicantAuthModel> result = applicantAuthRepo.findByEmail(request.getEmail());
    if (result.isEmpty()) {
        return ResponseEntity.status(NOT_FOUND).body(...);
    }
    
    ApplicantAuthModel user = result.get();
    
    // 2. Check if active
    if (!user.getIsActive()) {
        return ResponseEntity.status(UNAUTHORIZED).body(...);
    }
    
    // 3. Validate password
    if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
        return ResponseEntity.status(UNAUTHORIZED).body(...);
    }
    
    // 4. Create token
    String token = jwtService.createJweToken(
        new CreateTokenDto(user.getId(), user.getEmail(), user.getRole())
    );
    
    return ResponseEntity.ok(AuthResponse.builder().token(token).build());
}
```

---

### 3. **Google OAuth2 Flow**

```mermaid
sequenceDiagram
    participant Client
    participant Controller
    participant Service
    participant Google
    participant Database
    participant ProfileService

    Client->>Controller: GET /api/auth/google/url
    Controller-->>Client: Return Google OAuth URL
    Client->>Google: Redirect to Google Login
    Google-->>Client: Redirect with authorization code
    Client->>Controller: GET /api/auth/oauth2/callback/google?code=...
    Controller->>Service: googleLogin(code)
    Service->>Google: Exchange code for access token
    Google-->>Service: Access token
    Service->>Google: Get user info
    Google-->>Service: {email, sub, email_verified}
    Service->>Database: Check if user exists
    alt User exists
        Service->>Service: Update login info
    else New user
        Service->>Database: Create user (GOOGLE auth type)
        Service->>ProfileService: Create profile via Kafka
    end
    Service->>Service: Generate JWT token
    Service-->>Client: Return token + user info
```

**Code** ([ApplicantAuthInternalServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthInternalServiceImpl.java#L188)):

```java
public ResponseEntity<AuthResponse> googleLogin(String code) {
    // 1. Exchange code for access token
    String accessToken = googleOAuth2Service.getAccessToken(code);
    
    // 2. Get user info from Google
    GoogleUserInfo googleUserInfo = googleOAuth2Service.getUserInfo(accessToken);
    
    // 3. Check if user exists
    Optional<ApplicantAuthModel> existingUser = 
        applicantAuthRepo.findByEmail(googleUserInfo.getEmail());
    
    ApplicantAuthModel user;
    if (existingUser.isPresent()) {
        // Existing user - just update
        user = existingUser.get();
    } else {
        // New user - create account + profile
        user = applicantAuthRepo.save(
            ApplicantAuthModel.builder()
                .id(UUID.randomUUID())
                .email(googleUserInfo.getEmail())
                .role("APPLICANT")
                .authType("GOOGLE")
                .isVerified(true) // Google emails are pre-verified
                .isActive(true)
                .build()
        );
        
        // Create profile via Kafka
        eventProducer.createApplicantProfile(...);
    }
    
    // 4. Generate token
    String token = jwtService.createJweToken(
        new CreateTokenDto(user.getId(), user.getEmail(), user.getRole())
    );
    
    return ResponseEntity.ok(AuthResponse.builder().token(token).build());
}
```

**Google OAuth2 Service** ([GoogleOAuth2Service.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/oauth2/GoogleOAuth2Service.java)):
- Exchanges authorization code for access token
- Fetches user profile from Google API
- Returns standardized `GoogleUserInfo` DTO

---

### 4. **Request Authentication Flow**

Every authenticated request passes through [RequestFilter](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/RequestFilter.java):

```mermaid
graph TD
    A[Incoming Request] --> B{Has Authorization Header?}
    B -->|No| C[Continue without auth]
    B -->|Yes| D{Starts with 'Bearer '?}
    D -->|No| C
    D -->|Yes| E[Extract JWE token]
    E --> F[Decrypt JWE → get JWS]
    F --> G[Verify JWS signature]
    G --> H[Parse claims]
    H --> I[Add user data to request attributes]
    I --> J[Continue to controller]
    
    F -->|Decryption fails| C
    G -->|Signature invalid| C
    H -->|Token expired| C
```

**Code**:
```java
@Component
public class RequestFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            try {
                String jweToken = authHeader.substring(7);
                
                // Decrypt JWE to get JWS
                String jwsToken = jwtService.decryptJweToken(jweToken);
                
                // Parse and validate JWS
                ParseTokenDto tokenData = jwtService.parseJwsToken(jwsToken);
                
                // Add user info to request
                request.setAttribute("id", tokenData.getId());
                request.setAttribute("email", tokenData.getEmail());
                request.setAttribute("role", tokenData.getRole());
            } catch (Exception e) {
                log.warn("Invalid token: {}", e.getMessage());
            }
        }
        
        filterChain.doFilter(request, response);
    }
}
```

**Key Points**:
- Filter runs on **every request**
- Adds user context to request attributes if token is valid
- **Does NOT block** invalid tokens (Spring Security handles that)
- Controllers can access user data: `request.getAttribute("email")`

---

## 🛣️ Route Configuration

### How Routes Work

Routes are defined in [ApplicantAuthController.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthController.java) and protected by [SecurityConfig.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/SecurityConfig.java).

**Base Path**: `/api/auth`

### Route Protection Matrix

| Endpoint | Method | Authentication Required | Description |
|----------|--------|-------------------------|-------------|
| `/api/auth/greeting` | GET | ❌ No | Health check |
| `/api/auth/register` | POST | ❌ No | User registration |
| `/api/auth/login` | POST | ❌ No | User login |
| `/api/auth/validate` | GET | ❌ No | Token validation |
| `/api/auth/google/url` | GET | ❌ No | Get Google OAuth URL |
| `/api/auth/oauth2/callback/google` | GET | ❌ No | OAuth2 callback |
| `/api/auth/test-email` | POST | ❌ No | Test email sending |
| `/api/auth/verify/{userId}` | PUT | ✅ Yes | Verify account |
| `/profile/**` | PUT | ✅ Yes | Profile updates |
| **All other routes** | * | ✅ Yes | Requires auth |

### Security Configuration

**File**: [SecurityConfig.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/SecurityConfig.java#L22-L50)

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) {
    http
        .csrf(csrf -> csrf.disable())  // Stateless API
        .sessionManagement(session -> 
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth
            // Specific authenticated routes
            .requestMatchers(HttpMethod.PUT, "/profile/**").authenticated()
            
            // Public routes
            .requestMatchers(
                "/api/auth/register",
                "/api/auth/login",
                "/api/auth/validate",
                "/api/auth/google/url",
                "/api/auth/oauth2/callback/google",
                "/swagger-ui/**",
                "/v3/api-docs/**"
            ).permitAll()
            
            // Default: all other routes require authentication
            .anyRequest().authenticated()
        )
        // Add JWT validation filter
        .addFilterBefore(requestFilter, UsernamePasswordAuthenticationFilter.class);
    
    return http.build();
}
```

---

## 📡 API Endpoints

### Authentication Endpoints

#### 1. **Register New User**
```http
POST /api/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!",
  "firstname": "John",
  "lastName": "Doe",
  "phone": "+1234567890",
  "country": "USA",
  "city": "New York",
  "address": "123 Main St",
  "objective": "Looking for opportunities"
}
```

**Response** (201 Created):
```json
{
  "token": "eyJhbGciOiJSU0EtT0FFUC0yNTYi...",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "APPLICANT",
  "message": "Registration successful"
}
```

#### 2. **Login**
```http
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "SecurePass123!"
}
```

**Response** (200 OK):
```json
{
  "token": "eyJhbGciOiJSU0EtT0FFUC0yNTYi...",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@example.com",
  "role": "APPLICANT",
  "message": "Login successful"
}
```

#### 3. **Validate Token**
```http
GET /api/auth/validate
Authorization: Bearer eyJhbGciOiJSU0EtT0FFUC0yNTYi...
```

**Response** (200 OK):
```json
true
```

#### 4. **Get Google OAuth URL**
```http
GET /api/auth/google/url
```

**Response**:
```json
{
  "url": "https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&response_type=code&scope=email+profile"
}
```

#### 5. **Google OAuth Callback**
```http
GET /api/auth/oauth2/callback/google?code=4/0AeanS0aA...
```

**Response** (200 OK):
```json
{
  "token": "eyJhbGciOiJSU0EtT0FFUC0yNTYi...",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "user@gmail.com",
  "role": "APPLICANT",
  "message": "Google login successful"
}
```

#### 6. **Verify Account**
```http
PUT /api/auth/verify/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer eyJhbGciOiJSU0EtT0FFUC0yNTYi...
```

**Response** (200 OK):
```json
true
```

### Swagger Documentation

Interactive API documentation available at:
- **Gateway**: `/api/applicant-auth/swagger-ui.html`
- **Direct**: `http://localhost:8091/swagger-ui.html`

---

## 🔄 Event-Driven Architecture

This service communicates with other microservices via **Apache Kafka**.

### Event Types

#### 1. **Request-Reply Pattern** (Synchronous)
Used for operations that need immediate feedback (e.g., profile creation).

**Example**: Create Applicant Profile
```java
// Sending side
CreateApplicantRes response = eventProducer.createApplicantProfile(
    new CreateApplicantReq(userId, email, firstname, ...)
);

// EventProducerImpl.java
private Object sendAndReceive(String reqTopic, String resTopic, Object data) {
    ProducerRecord<String, Object> req = new ProducerRecord<>(reqTopic, data);
    req.headers().add(KafkaHeaders.REPLY_TOPIC, resTopic.getBytes());
    
    return replyingKafkaTemplate.sendAndReceive(req)
        .get(replyTimeout, TimeUnit.MILLISECONDS)
        .value();
}
```

**Topics**:
- Request: `applicant-profile.save.req`
- Response: `applicant-profile.save.res`

**Timeout**: 5 seconds (configurable via `spring.kafka.reply-timeout`)

#### 2. **Fire-and-Forget Pattern** (Asynchronous)
Used for non-critical operations (e.g., sending emails).

**Example**: Send Activation Email
```java
// Sending side
eventProducer.sendAccountActivationEmail(
    new SendAccountActivationEmailReq(
        EmailRefModule.APPLICANT,
        userId,
        email,
        activationLink,
        expirationTime
    )
);

// EventProducerImpl.java
public void sendAccountActivationEmail(SendAccountActivationEmailReq req) {
    kafkaTemplate.send(EmailTopicRegistry.ACCOUNT_ACTIVATION, req);
}
```

**Topic**: `email.account-activation`

### Kafka Configuration

**Producer Config** ([KafkaProducerConfig.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/KafkaProducerConfig.java)):
```java
@Bean
public ProducerFactory<String, Object> replyProducerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    return new DefaultKafkaProducerFactory<>(props);
}

@Bean
ReplyingKafkaTemplate<String, Object, Object> replyingKafkaTemplate(
        ConcurrentMessageListenerContainer<String, Object> replyListenerContainer) {
    var template = new ReplyingKafkaTemplate<>(
        replyProducerFactory(), 
        replyListenerContainer
    );
    template.setDefaultReplyTimeout(Duration.ofSeconds(5));
    return template;
}
```

**application.yml**:
```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    reply-timeout: ${KAFKA_REPLY_TIMEOUT:10000}
```

### Event Producer Interface

**Location**: [EventProducer.java](applicant_auth_api/src/main/java/devision/wukong/applicant_auth_api/external/EventProducer.java)

```java
public interface EventProducer {
    // Synchronous - waits for response
    CreateApplicantRes createApplicantProfile(CreateApplicantReq req);
    
    // Asynchronous - fire and forget
    void sendAccountActivationEmail(SendAccountActivationEmailReq req);
}
```

### Error Handling

**Custom Exceptions**:
- `ProfileCreationException`: Profile creation failed → triggers transaction rollback
- `KafkaTimeoutException`: Kafka request timed out

**Transaction Rollback**:
```java
@Transactional(rollbackFor = Exception.class)
public ResponseEntity<AuthResponse> register(RegisterRequest request) {
    ApplicantAuthModel savedUser = applicantAuthRepo.save(...);
    
    try {
        eventProducer.createApplicantProfile(...);  // Sync call
    } catch (ProfileCreationException e) {
        // Transaction rolls back - user won't be saved
        throw new RuntimeException("Failed to create user profile", e);
    }
    
    // Email sending failure won't rollback
    try {
        eventProducer.sendAccountActivationEmail(...);
    } catch (Exception e) {
        log.warn("Failed to send activation email: {}", e.getMessage());
    }
}
```

---

## ⚙️ Configuration

### Environment Variables

Create a `.env` file in the project root:

```properties
# Database
POSTGRES_URI=jdbc:postgresql://localhost:5432/applicant_auth_db

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_REPLY_TIMEOUT=10000

# Eureka Service Discovery
EUREKA_SERVER_URI=http://localhost:8761/eureka

# Google OAuth2
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# KeyStore
KEYSTORE_FILENAME=keystore.p12
KEYSTORE_PWD=your-keystore-password
KEYSTORE_ALIAS_JWS=jws
KEYSTORE_ALIAS_JWE=jwe

# JWT
JWT_EXPIRATION=86400000

# Data Seeding (optional)
SEED_KEY=your-seed-key
```

### application.yml

**Location**: [application.yml](applicant_auth_service/src/main/resources/application.yml)

```yaml
server:
  port: 8091

spring:
  application:
    name: applicant-auth
  
  datasource:
    url: ${POSTGRES_URI}
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: update  # Auto-create/update tables
    show-sql: true
  
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

key-store:
  type: PKCS12
  path: classpath:${KEYSTORE_FILENAME}
  password: ${KEYSTORE_PWD}
  alias:
    jws: ${KEYSTORE_ALIAS_JWS}
    jwe: ${KEYSTORE_ALIAS_JWE}

jwt:
  expiration: ${JWT_EXPIRATION:86400000}

google:
  oauth2:
    client-id: ${GOOGLE_CLIENT_ID}
    client-secret: ${GOOGLE_CLIENT_SECRET}
    redirect-uri: http://localhost:8091/api/auth/oauth2/callback/google
    token-uri: https://oauth2.googleapis.com/token
    user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
```

---

## 🚀 Building & Running

### Prerequisites

- **Java 21** or higher
- **Maven 3.8+**
- **PostgreSQL** running on port 5432
- **Kafka** running on port 9092
- **Eureka Server** running on port 8761

### Local Development

#### Option 1: Using Make

```bash
# Build the project
make build

# Run the service
make run

# Run with database seeding
make run-seed
```

#### Option 2: Using Maven Wrapper

```bash
# Build
./mvnw package -DskipTests

# Run
java -jar applicant_auth_service/target/applicant_auth_service-0.0.1-SNAPSHOT.jar
```

#### Option 3: Using Docker

```bash
# Build Docker image
docker build \
  --build-arg GITHUB_USERNAME=your-username \
  --build-arg GITHUB_TOKEN=your-token \
  --build-arg MY_MODULE=applicant_auth \
  -t applicant-auth-service .

# Run container
docker run -p 8091:8091 \
  -e POSTGRES_URI=jdbc:postgresql://host.docker.internal:5432/applicant_auth_db \
  -e KAFKA_BOOTSTRAP_SERVERS=host.docker.internal:9092 \
  -e EUREKA_SERVER_URI=http://host.docker.internal:8761/eureka \
  applicant-auth-service
```

### Makefile Commands

```makefile
# Check Java version
make version

# Build the service module
make build

# Build and run
make run

# Build and run with data seeding
make run-seed
```

---

## 🔧 Modifying Security

### Adding a New Security Rule

**File**: [SecurityConfig.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/SecurityConfig.java)

#### Example 1: Make a Route Public

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(
        "/api/auth/register",
        "/api/auth/login",
        "/api/auth/your-new-public-endpoint"  // Add this
    ).permitAll()
    .anyRequest().authenticated()
)
```

#### Example 2: Require Authentication for Specific Routes

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(HttpMethod.PUT, "/profile/**").authenticated()
    .requestMatchers(HttpMethod.DELETE, "/admin/**").authenticated()  // Add this
    .requestMatchers("/api/auth/register", ...).permitAll()
    .anyRequest().authenticated()
)
```

#### Example 3: Role-Based Access Control

To add role-based authorization:

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/admin/**")
        .hasRole("ADMIN")  // Only ADMIN role
    .requestMatchers("/api/auth/user/**")
        .hasAnyRole("USER", "ADMIN")  // USER or ADMIN
    .requestMatchers("/api/auth/register", ...).permitAll()
    .anyRequest().authenticated()
)
```

**Note**: Roles in JWT should be stored as "ADMIN" not "ROLE_ADMIN" (Spring adds the prefix automatically).

### Changing JWT Expiration

**File**: [JwtServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/security/JwtServiceImpl.java)

```java
// Current: 1 day
private final long jwsExpiration = 1000L * 60 * 60 * 24 * 1;

// Change to 7 days
private final long jwsExpiration = 1000L * 60 * 60 * 24 * 7;

// Or make it configurable via application.yml
@Value("${jwt.expiration}")
private long jwsExpiration;
```

### Changing Encryption Algorithms

**File**: [JwtServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/security/JwtServiceImpl.java)

```java
// Current algorithms
private final SignatureAlgorithm signatureAlgorithm = Jwts.SIG.RS256;
private final KeyAlgorithm<PublicKey, PrivateKey> encryptionKeyAlgorithm = Jwts.KEY.RSA_OAEP_256;
private final AeadAlgorithm encryptionAlgorithm = Jwts.ENC.A256GCM;

// Alternative options:
// Signature: RS384, RS512, PS256, PS384, PS512
// Key encryption: RSA_OAEP, RSA_OAEP_256, RSA_OAEP_384, RSA_OAEP_512
// Content encryption: A128GCM, A192GCM, A256GCM
```

### Generating New KeyStore

```bash
# Generate JWS keypair
keytool -genkeypair -alias jws \
  -keyalg RSA -keysize 2048 \
  -storetype PKCS12 \
  -keystore keystore.p12 \
  -storepass your-password \
  -validity 365

# Generate JWE keypair
keytool -genkeypair -alias jwe \
  -keyalg RSA -keysize 2048 \
  -storetype PKCS12 \
  -keystore keystore.p12 \
  -storepass your-password \
  -validity 365

# List aliases
keytool -list -keystore keystore.p12 -storepass your-password
```

Place the generated `keystore.p12` in [src/main/resources/](applicant_auth_service/src/main/resources/).

### Disabling JWE (Use Only JWS)

If you want simpler tokens without encryption:

**In Controllers**:
```java
// Change from:
String token = jwtService.createJweToken(tokenDto);

// To:
String token = jwtService.createJwsToken(tokenDto);
```

**In RequestFilter**:
```java
// Change from:
String jwsToken = jwtService.decryptJweToken(jweToken);
ParseTokenDto tokenData = jwtService.parseJwsToken(jwsToken);

// To:
ParseTokenDto tokenData = jwtService.parseJwsToken(jweToken);
```

**⚠️ Warning**: This reduces security - tokens can be read by anyone who intercepts them.

---

## 🛣️ Modifying Routes

### Adding a New Endpoint

#### Step 1: Define DTO (if needed)

**Location**: [applicant_auth_api/internal/dto/](applicant_auth_api/src/main/java/devision/wukong/applicant_auth_api/internal/dto/)

```java
package devision.wukong.applicant_auth_api.internal.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class PasswordResetRequest {
    private String email;
    private String newPassword;
}
```

#### Step 2: Add Method to Service Interface

**File**: [ApplicantAuthInternalService.java](applicant_auth_api/src/main/java/devision/wukong/applicant_auth_api/internal/ApplicantAuthInternalService.java)

```java
public interface ApplicantAuthInternalService {
    ResponseEntity<AuthResponse> register(RegisterRequest request) throws Exception;
    ResponseEntity<AuthResponse> login(LoginRequest request);
    ResponseEntity<Boolean> validateToken(String authHeader);
    
    // Add new method
    ResponseEntity<Boolean> resetPassword(PasswordResetRequest request);
}
```

#### Step 3: Implement Business Logic

**File**: [ApplicantAuthInternalServiceImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthInternalServiceImpl.java)

```java
@Override
public ResponseEntity<Boolean> resetPassword(PasswordResetRequest request) {
    Optional<ApplicantAuthModel> userOpt = applicantAuthRepo.findByEmail(request.getEmail());
    
    if (userOpt.isEmpty()) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(false);
    }
    
    ApplicantAuthModel user = userOpt.get();
    user.setPasswordHash(passwordEncoder.encode(request.getNewPassword()));
    applicantAuthRepo.save(user);
    
    return ResponseEntity.ok(true);
}
```

#### Step 4: Add Controller Endpoint

**File**: [ApplicantAuthController.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthController.java)

```java
@Operation(summary = "Reset user password", description = "Reset password for a user account")
@ApiResponses(value = {
    @ApiResponse(responseCode = "200", description = "Password reset successfully"),
    @ApiResponse(responseCode = "404", description = "User not found")
})
@PostMapping("/reset-password")
public ResponseEntity<Boolean> resetPassword(
    @Parameter(description = "Password reset request", required = true) 
    @RequestBody PasswordResetRequest request) {
    return applicantAuthInternalService.resetPassword(request);
}
```

#### Step 5: Configure Security (if needed)

**File**: [SecurityConfig.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/SecurityConfig.java)

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers(
        "/api/auth/register",
        "/api/auth/login",
        "/api/auth/reset-password"  // Add this for public access
    ).permitAll()
    .anyRequest().authenticated()
)
```

### Changing Base Path

**Current**: `/api/auth`

To change to `/v1/auth`:

**File**: [ApplicantAuthController.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthController.java)

```java
// Change from:
@RequestMapping("/api/auth")

// To:
@RequestMapping("/v1/auth")
```

**Update SecurityConfig**:
```java
.requestMatchers(
    "/v1/auth/register",
    "/v1/auth/login",
    // ... other routes
).permitAll()
```

**Update Google Redirect URI**:
```yaml
google:
  oauth2:
    redirect-uri: http://localhost:8091/v1/auth/oauth2/callback/google
```

---

## 🧩 Key Components Explained

### 1. **ApplicantAuthModel** - User Entity

**File**: [ApplicantAuthModel.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/applicant_auth/ApplicantAuthModel.java)

```java
@Entity(name = "applicant_auth")
public class ApplicantAuthModel {
    @Id
    private UUID id;                    // Primary key
    
    @Column(unique = true)
    private String email;               // Unique email
    
    private String passwordHash;        // BCrypt hashed password (null for OAuth)
    private Boolean isVerified;         // Email verified
    private Boolean isActive;           // Account active
    private String role;                // User role (APPLICANT)
    private String authType;            // LOCAL or GOOGLE
    private String externalProvider;    // OAuth provider name
    private String externalProviderToken; // OAuth access token
}
```

**Database Table**: `applicant_auth`

**Indexes**: Unique index on `email`

### 2. **JwtService** - Token Management

**Interface**: [JwtService.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/security/JwtService.java)

```java
public interface JwtService {
    // Create signed token (JWS)
    String createJwsToken(CreateTokenDto dto) throws Exception;
    
    // Parse and validate signed token
    ParseTokenDto parseJwsToken(String token) throws Exception;
    
    // Create encrypted token (JWE wrapping JWS)
    String createJweToken(CreateTokenDto dto) throws Exception;
    
    // Decrypt JWE to get JWS
    String decryptJweToken(String jwe) throws Exception;
}
```

**Token Creation Flow**:
```
User Data → CreateTokenDto → JWS (signed) → JWE (encrypted) → Client
```

**Token Validation Flow**:
```
Client Token → JWE (decrypt) → JWS (verify) → ParseTokenDto → User Data
```

### 3. **KeyService** - Cryptographic Key Management

**Interface**: [KeyService.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/security/KeyService.java)

```java
public interface KeyService {
    PrivateKey getJwsPrivateKey();   // Sign JWS tokens
    PublicKey getJwsPublickey();     // Verify JWS tokens
    PrivateKey getJwePrivateKey();   // Decrypt JWE tokens
    PublicKey getJwePublicKey();     // Encrypt JWE tokens
}
```

**Lazy Loading**: KeyStore loaded once on first access, then cached.

### 4. **EventProducer** - Kafka Integration

**Interface**: [EventProducer.java](applicant_auth_api/src/main/java/devision/wukong/applicant_auth_api/external/EventProducer.java)

```java
public interface EventProducer {
    // Synchronous - waits for acknowledgment
    CreateApplicantRes createApplicantProfile(CreateApplicantReq req);
    
    // Asynchronous - fire and forget
    void sendAccountActivationEmail(SendAccountActivationEmailReq req);
}
```

**Implementation**: [EventProducerImpl.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/kafka/EventProducerImpl.java)

### 5. **RequestFilter** - JWT Validation Filter

**File**: [RequestFilter.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/config/RequestFilter.java)

Runs **before** every request to validate JWT tokens.

**Flow**:
1. Extract `Authorization` header
2. Check if starts with "Bearer "
3. Extract token
4. Decrypt JWE → get JWS
5. Verify JWS signature and expiration
6. Parse user claims
7. Add user data to request attributes
8. Continue to controller

**Access User Data in Controller**:
```java
@GetMapping("/profile")
public ResponseEntity<?> getProfile(HttpServletRequest request) {
    String userId = (String) request.getAttribute("id");
    String email = (String) request.getAttribute("email");
    String role = (String) request.getAttribute("role");
    
    // Use user data
}
```

### 6. **GoogleOAuth2Service** - OAuth2 Integration

**File**: [GoogleOAuth2Service.java](applicant_auth_service/src/main/java/devision/wukong/applicant_auth_service/common/oauth2/GoogleOAuth2Service.java)

**Methods**:
```java
public String getAccessToken(String code) {
    // Exchange authorization code for access token
    // POST https://oauth2.googleapis.com/token
}

public GoogleUserInfo getUserInfo(String accessToken) {
    // Fetch user profile from Google
    // GET https://www.googleapis.com/oauth2/v3/userinfo
}
```

**Configuration**: Uses `application.yml` settings for client ID, secret, and redirect URI.

---

## 📊 Database Schema

### Table: `applicant_auth`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Unique user ID |
| `email` | VARCHAR | UNIQUE, NOT NULL | User email |
| `password_hash` | VARCHAR | NULLABLE | BCrypt password hash (null for OAuth) |
| `is_verified` | BOOLEAN | NOT NULL | Email verification status |
| `is_active` | BOOLEAN | NOT NULL | Account active status |
| `role` | VARCHAR | NOT NULL | User role (APPLICANT) |
| `auth_type` | VARCHAR | NOT NULL | LOCAL or GOOGLE |
| `external_provider` | VARCHAR | NULLABLE | OAuth provider name |
| `external_provider_token` | VARCHAR | NULLABLE | OAuth access token |

**Hibernate DDL**: `spring.jpa.hibernate.ddl-auto=update` (auto-creates tables)

---

## 🧪 Testing

### Test Email Sending

```bash
curl -X POST "http://localhost:8091/api/auth/test-email?email=test@example.com"
```

### Test Token Validation

```bash
# Login first
TOKEN=$(curl -X POST http://localhost:8091/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}' \
  | jq -r '.token')

# Validate token
curl -X GET http://localhost:8091/api/auth/validate \
  -H "Authorization: Bearer $TOKEN"
```

### Test Google OAuth Flow

1. Get OAuth URL:
```bash
curl http://localhost:8091/api/auth/google/url
```

2. Visit the URL in browser
3. Copy the `code` parameter from callback URL
4. Exchange code for token:
```bash
curl "http://localhost:8091/api/auth/oauth2/callback/google?code=YOUR_CODE"
```

---

## 🔍 Debugging

### Enable Debug Logging

**application.yml**:
```yaml
logging:
  level:
    "[devision.wukong]": DEBUG
    "[org.springframework.security]": DEBUG
    "[org.springframework.kafka]": DEBUG
```

### Common Issues

#### 1. **KeyStore not found**
```
Failed to load keystore from classpath:keystore.p12
```

**Solution**: 
- Ensure `keystore.p12` exists in `src/main/resources/`
- Check `KEYSTORE_FILENAME` environment variable

#### 2. **Token decryption failed**
```
Invalid token: Unable to decrypt content
```

**Solution**:
- Verify KeyStore has correct JWE keypair
- Check alias name matches `KEYSTORE_ALIAS_JWE`

#### 3. **Kafka timeout**
```
KafkaTimeoutException: Kafka timeout after 5000ms
```

**Solution**:
- Check Kafka is running: `nc -zv localhost 9092`
- Increase timeout: `spring.kafka.reply-timeout=10000`
- Verify consumer service is running

#### 4. **Google OAuth redirect mismatch**
```
redirect_uri_mismatch
```

**Solution**:
- Update redirect URI in Google Cloud Console
- Ensure it matches `application.yml`: `http://localhost:8091/api/auth/oauth2/callback/google`

---

## 🚀 Deployment

### Docker Compose Example

```yaml
version: '3.8'

services:
  applicant-auth:
    build:
      context: .
      args:
        GITHUB_USERNAME: ${GITHUB_USERNAME}
        GITHUB_TOKEN: ${GITHUB_TOKEN}
        MY_MODULE: applicant_auth
    ports:
      - "8091:8091"
    environment:
      POSTGRES_URI: jdbc:postgresql://postgres:5432/applicant_auth_db
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_SERVER_URI: http://eureka:8761/eureka
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      KEYSTORE_FILENAME: keystore.p12
      KEYSTORE_PWD: ${KEYSTORE_PWD}
      KEYSTORE_ALIAS_JWS: jws
      KEYSTORE_ALIAS_JWE: jwe
    depends_on:
      - postgres
      - kafka
      - eureka

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: applicant_auth_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"

  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    ports:
      - "9092:9092"
```

---

## 📚 Additional Resources

- **Spring Security**: https://docs.spring.io/spring-security/reference/
- **JJWT Library**: https://github.com/jwtk/jjwt
- **Spring Kafka**: https://docs.spring.io/spring-kafka/reference/
- **Google OAuth2**: https://developers.google.com/identity/protocols/oauth2

---

## 📝 Summary

This applicant authentication service provides:

✅ **Secure Authentication**: Multi-layer JWT (JWE + JWS) with RSA encryption  
✅ **Multiple Auth Methods**: Email/password and Google OAuth2  
✅ **Stateless Architecture**: No server-side sessions  
✅ **Event-Driven**: Kafka integration for profile creation and email notifications  
✅ **Microservice Ready**: Eureka service discovery, Docker support  
✅ **Well Documented**: OpenAPI/Swagger interactive docs  
✅ **Production Ready**: Transaction management, error handling, logging  

For questions or issues, contact the Devision Wukong team.
