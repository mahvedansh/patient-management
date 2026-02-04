# Backend Architecture Guide: Java Spring Boot & gRPC Patient Management System

## Table of Contents
1. [Project Overview](#project-overview)
2. [Microservices Architecture](#microservices-architecture)
3. [Technology Stack](#technology-stack)
4. [Patient Service Deep Dive](#patient-service-deep-dive)
5. [Billing Service Deep Dive](#billing-service-deep-dive)
6. [Inter-Service Communication](#inter-service-communication)
7. [Data Flow & Architecture Patterns](#data-flow--architecture-patterns)
8. [Configuration Management](#configuration-management)
9. [Deployment & Docker](#deployment--docker)
10. [API Examples & Testing](#api-examples--testing)
11. [Development Workflow](#development-workflow)

---

## Project Overview

This is a **microservices-based patient management system** built with **Java Spring Boot** and **gRPC**. The system consists of two main microservices that work together to manage patient data and billing accounts.

### Key Features
- **Patient Management**: CRUD operations for patient records
- **Billing Integration**: Automatic billing account creation via gRPC
- **Data Validation**: Comprehensive input validation with custom groups
- **Error Handling**: Global exception handling with custom exceptions
- **Database Integration**: PostgreSQL with JPA/Hibernate
- **RESTful APIs**: Spring Web MVC with OpenAPI documentation
- **Docker Support**: Multi-stage Docker builds for both services

---

## Microservices Architecture

```
┌─────────────────┐    gRPC    ┌─────────────────┐
│  Patient        │────────────│  Billing       │
│  Service        │            │  Service       │
│                 │            │                 │
│  • REST API     │            │  • gRPC Server │
│  • JPA/Hibernate│            │  • Business    │
│  • PostgreSQL   │            │    Logic       │
│  • Port: 4000   │            │  • Port: 4101  │
└─────────────────┘            └─────────────────┘
         │                              │
         └─────────────┬────────────────┘
                       │
                ┌─────────────┐
                │ PostgreSQL  │
                │ Database    │
                │ Port: 5432  │
                └─────────────┘
```

### Service Responsibilities

**Patient Service:**
- Exposes RESTful APIs for patient CRUD operations
- Manages patient data persistence
- Validates patient data
- Communicates with Billing Service via gRPC

**Billing Service:**
- Provides gRPC service for billing account management
- Handles billing account creation logic
- Returns billing account status

---

## Technology Stack

### Core Technologies
- **Java 21**: Latest LTS version with modern features
- **Spring Boot 3.4.0**: Framework for building microservices
- **Spring Data JPA**: Data access layer
- **PostgreSQL**: Primary database
- **gRPC**: Inter-service communication
- **Protocol Buffers**: Interface definition language

### Spring Boot Starters
- `spring-boot-starter-data-jpa`: Database operations
- `spring-boot-starter-validation`: Input validation
- `spring-boot-starter-web`: REST API development
- `spring-boot-starter-webmvc`: MVC pattern
- `spring-boot-devtools`: Development tools
- `springdoc-openapi-starter-webmvc-ui`: API documentation

### Build & Deployment
- **Maven**: Dependency management and build tool
- **Docker**: Containerization with multi-stage builds
- **JUnit**: Unit testing framework

---

## Patient Service Deep Dive

### Project Structure
```
patient-service/
├── src/main/java/com/pm/patientservice/
│   ├── PatientServiceApplication.java          # Main Spring Boot Application
│   ├── controller/
│   │   └── PatientController.java              # REST API Endpoints
│   ├── service/
│   │   └── PatientService.java                 # Business Logic Layer
│   ├── repository/
│   │   └── PatientRepository.java              # Data Access Layer
│   ├── model/
│   │   └── Patient.java                        # JPA Entity
│   ├── dto/
│   │   ├── PatientRequestDTO.java              # Input DTO
│   │   ├── PatientResponseDTO.java             # Output DTO
│   │   └── validators/
│   │       └── CreatePatientValidationGroup.java # Validation Groups
│   ├── mapper/
│   │   └── PatientMapper.java                  # Object Mapping
│   ├── exception/
│   │   ├── GlobalExceptionHandler.java         # Global Exception Handling
│   │   ├── EmailAlreadyExistsException.java    # Custom Exceptions
│   │   └── PatientNotFoundException.java
│   ├── grpc/
│   │   └── BillingServiceGrpcClient.java       # gRPC Client
│   └── PatientServiceApplication.java
├── src/main/proto/
│   ├── billing_service.proto                   # gRPC Contracts
│   └── patient_event.proto                     # Event Definitions
├── src/main/resources/
│   ├── application.properties                  # Configuration
│   └── data.sql                               # Initial Data
├── Dockerfile                                  # Docker Configuration
└── pom.xml                                     # Maven Configuration
```

### 1. Main Application Class

```java
@SpringBootApplication
public class PatientServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PatientServiceApplication.class, args);
    }
}
```

**Annotations:**
- `@SpringBootApplication`: Combines `@Configuration`, `@EnableAutoConfiguration`, `@ComponentScan`

### 2. Data Model (JPA Entity)

```java
@Entity
public class Patient {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private UUID id;

    @NotNull
    private String name;

    @NotNull
    @Email
    @Column(unique = true)
    private String email;

    @NotNull
    private String address;

    @NotNull
    private LocalDate dateOfBirth;

    @NotNull
    private LocalDate registeredDate;

    // Getters and Setters...
}
```

**Key Features:**
- **UUID Primary Key**: Auto-generated unique identifiers
- **Email Uniqueness**: Database-level constraint
- **Validation Annotations**: Bean validation constraints
- **LocalDate Fields**: Proper date handling

### 3. Data Transfer Objects (DTOs)

**PatientRequestDTO:**
```java
public class PatientRequestDTO {
    @NotBlank(message = "Name is required")
    @Size(max = 100, message = "Name cannot exceed 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email should be valid")
    private String email;

    @NotBlank(message = "Address is required")
    private String address;

    @NotBlank(message = "Date of birth is required")
    private String dateOfBirth;

    @NotBlank(groups = CreatePatientValidationGroup.class,
              message = "Registered date is required")
    private String registeredDate;

    // Getters and Setters...
}
```

**PatientResponseDTO:**
```java
public class PatientResponseDTO {
    private String id;
    private String name;
    private String email;
    private String address;
    private String dateOfBirth;
    // Getters and Setters...
}
```

**Validation Groups:**
```java
public interface CreatePatientValidationGroup {
    // Marker interface for validation groups
}
```

### 4. Repository Layer

```java
@Repository
public interface PatientRepository extends JpaRepository<Patient, UUID> {
    boolean existsByEmail(String email);
    boolean existsByEmailAndIdNot(String email, UUID id);
}
```

**Methods:**
- `existsByEmail()`: Check email uniqueness for new patients
- `existsByEmailAndIdNot()`: Check email uniqueness excluding current patient (for updates)

### 5. Service Layer (Business Logic)

```java
@Service
public class PatientService {

    private final PatientRepository patientRepository;
    private final BillingServiceGrpcClient billingServiceGrpcClient;

    public PatientService(PatientRepository patientRepository,
                         BillingServiceGrpcClient billingServiceGrpcClient) {
        this.patientRepository = patientRepository;
        this.billingServiceGrpcClient = billingServiceGrpcClient;
    }

    public List<PatientResponseDTO> getPatients() {
        List<Patient> patients = patientRepository.findAll();
        return patients.stream()
                .map(PatientMapper::toDTO)
                .toList();
    }

    public PatientResponseDTO createPatient(PatientRequestDTO patientRequestDTO) {
        // Email uniqueness validation
        if(patientRepository.existsByEmail(patientRequestDTO.getEmail())) {
            throw new EmailAlreadyExistsException(
                "A patient with this email already exists: " +
                patientRequestDTO.getEmail());
        }

        // Save patient
        Patient newPatient = patientRepository.save(
            PatientMapper.toModel(patientRequestDTO));

        // Create billing account via gRPC
        billingServiceGrpcClient.createBillingAccount(
            newPatient.getId().toString(),
            newPatient.getName(),
            newPatient.getEmail());

        return PatientMapper.toDTO(newPatient);
    }

    public PatientResponseDTO updatePatient(UUID id, PatientRequestDTO patientRequestDTO) {
        // Find existing patient
        Patient patient = patientRepository.findById(id)
            .orElseThrow(() -> new PatientNotFoundException(
                "Patient not found with ID: " + id));

        // Email uniqueness validation (exclude current patient)
        if (patientRepository.existsByEmailAndIdNot(patientRequestDTO.getEmail(), id)){
            throw new EmailAlreadyExistsException(
                "A patient with this email already exists: " +
                patientRequestDTO.getEmail());
        }

        // Update patient fields
        patient.setName(patientRequestDTO.getName());
        patient.setAddress(patientRequestDTO.getAddress());
        patient.setEmail(patientRequestDTO.getEmail());
        patient.setDateOfBirth(LocalDate.parse(patientRequestDTO.getDateOfBirth()));

        Patient updatedPatient = patientRepository.save(patient);
        return PatientMapper.toDTO(updatedPatient);
    }

    public void deletePatient(UUID id) {
        if (!patientRepository.existsById(id)) {
            throw new PatientNotFoundException("Patient not found with ID: " + id);
        }
        patientRepository.deleteById(id);
    }
}
```

**Key Business Logic:**
1. **Email Uniqueness**: Prevents duplicate email addresses
2. **Billing Integration**: Automatically creates billing account for new patients
3. **Optimistic Updates**: Validates existence before updates/deletes
4. **Exception Handling**: Throws custom exceptions for business rule violations

### 6. Controller Layer (REST API)

```java
@RestController
@RequestMapping("/patients")
@Tag(name = "Patient", description = "API for managing Patients")
public class PatientController {

    private final PatientService patientService;

    public PatientController(PatientService patientService) {
        this.patientService = patientService;
    }

    @GetMapping
    @Operation(summary = "Get Patients")
    public ResponseEntity<List<PatientResponseDTO>> getPatients() {
        List<PatientResponseDTO> patients = patientService.getPatients();
        return ResponseEntity.ok().body(patients);
    }

    @PostMapping
    @Operation(summary = "Create a new Patient")
    public ResponseEntity<PatientResponseDTO> createPatient(
            @Validated({Default.class, CreatePatientValidationGroup.class})
            @RequestBody PatientRequestDTO patientRequestDTO) {

        PatientResponseDTO patientResponseDTO = patientService.createPatient(
                patientRequestDTO);

        return ResponseEntity.ok().body(patientResponseDTO);
    }

    @PutMapping("/{id}")
    @Operation(summary = "Update a new Patient")
    public ResponseEntity<PatientResponseDTO> updatePatient(
            @PathVariable UUID id,
            @Validated({Default.class})
            @RequestBody PatientRequestDTO patientRequestDTO) {

        PatientResponseDTO patientResponseDTO = patientService.updatePatient(id,
                patientRequestDTO);

        return ResponseEntity.ok().body(patientResponseDTO);
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Delete a Patient")
    public ResponseEntity<Void> deletePatient(@PathVariable UUID id) {
        patientService.deletePatient(id);
        return ResponseEntity.noContent().build();
    }
}
```

**API Endpoints:**
- `GET /patients`: Retrieve all patients
- `POST /patients`: Create new patient
- `PUT /patients/{id}`: Update existing patient
- `DELETE /patients/{id}`: Delete patient

**Validation Groups:**
- **Create Operation**: Uses both `Default` and `CreatePatientValidationGroup`
- **Update Operation**: Uses only `Default` (registeredDate not required)

### 7. Object Mapping

```java
public class PatientMapper {
    public static PatientResponseDTO toDTO(Patient patient) {
        PatientResponseDTO patientDTO = new PatientResponseDTO();
        patientDTO.setId(patient.getId().toString());
        patientDTO.setName(patient.getName());
        patientDTO.setAddress(patient.getAddress());
        patientDTO.setEmail(patient.getEmail());
        patientDTO.setDateOfBirth(patient.getDateOfBirth().toString());
        return patientDTO;
    }

    public static Patient toModel(PatientRequestDTO patientRequestDTO) {
        Patient patient = new Patient();
        patient.setName(patientRequestDTO.getName());
        patient.setAddress(patientRequestDTO.getAddress());
        patient.setEmail(patientRequestDTO.getEmail());
        patient.setDateOfBirth(LocalDate.parse(patientRequestDTO.getDateOfBirth()));
        patient.setRegisteredDate(LocalDate.parse(patientRequestDTO.getRegisteredDate()));
        return patient;
    }
}
```

**Mapping Strategy:**
- **Entity → DTO**: Converts database entities to API response format
- **DTO → Entity**: Converts API requests to database entities
- **Type Conversion**: String dates ↔ LocalDate objects

### 8. Exception Handling

**Custom Exceptions:**
```java
public class EmailAlreadyExistsException extends RuntimeException {
    public EmailAlreadyExistsException(String message) {
        super(message);
    }
}

public class PatientNotFoundException extends RuntimeException {
    public PatientNotFoundException(String message) {
        super(message);
    }
}
```

**Global Exception Handler:**
```java
@ControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(
            GlobalExceptionHandler.class);

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationException(
            MethodArgumentNotValidException ex) {

        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(
                error -> errors.put(error.getField(), error.getDefaultMessage()));

        return ResponseEntity.badRequest().body(errors);
    }

    @ExceptionHandler(EmailAlreadyExistsException.class)
    public ResponseEntity<Map<String, String>> handleEmailAlreadyExistsException(
            EmailAlreadyExistsException ex) {

        log.warn("Email address already exist {}", ex.getMessage());
        Map<String, String> errors = new HashMap<>();
        errors.put("message", "Email address already exists");
        return ResponseEntity.badRequest().body(errors);
    }

    @ExceptionHandler(PatientNotFoundException.class)
    public ResponseEntity<Map<String, String>> handlePatientNotFoundException(
            PatientNotFoundException ex) {
        log.warn("Patient not found {}", ex.getMessage());

        Map<String, String> errors = new HashMap<>();
        errors.put("message", "Patient not found");
        return ResponseEntity.badRequest().body(errors);
    }
}
```

**Exception Handling Strategy:**
- **Validation Errors**: Field-level validation errors with detailed messages
- **Business Rule Violations**: Custom exceptions with meaningful messages
- **Logging**: Warn-level logging for business exceptions
- **HTTP Status Codes**: Appropriate status codes (400 for client errors)

---

## Billing Service Deep Dive

### Project Structure
```
billing-service/
├── src/main/java/com/pm/billingservice/
│   ├── BillingServiceApplication.java          # Main Spring Boot Application
│   └── grpc/
│       └── BillingGrpcService.java             # gRPC Service Implementation
├── src/main/proto/
│   └── billing_service.proto                   # gRPC Contract Definition
├── src/main/resources/
│   └── application.properties                  # Configuration
├── Dockerfile                                  # Docker Configuration
└── pom.xml                                     # Maven Configuration
```

### 1. Main Application Class

```java
@SpringBootApplication
public class BillingServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(BillingServiceApplication.class, args);
    }
}
```

### 2. gRPC Service Implementation

```java
@GrpcService
public class BillingGrpcService extends BillingServiceImplBase {

    private static final Logger log = LoggerFactory.getLogger(
            BillingGrpcService.class);

    @Override
    public void createBillingAccount(BillingRequest billingRequest,
                                     StreamObserver<BillingResponse> responseObserver) {

        log.info("createBillingAccount request received {}", billingRequest.toString());

        // Business logic - e.g save to database, perform calculates etc

        BillingResponse response = BillingResponse.newBuilder()
                .setAccountId("12345")
                .setStatus("ACTIVE")
                .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

**gRPC Implementation Details:**
- **Extends BillingServiceImplBase**: Auto-generated base class from proto file
- **Streaming Response**: Uses `StreamObserver` for asynchronous responses
- **Business Logic**: Currently returns mock data (would integrate with billing system)
- **Logging**: Request logging for debugging

### 3. Protocol Buffer Definition

```protobuf
syntax = "proto3";

option java_multiple_files = true;
option java_package = "billing";

service BillingService {
  rpc CreateBillingAccount (BillingRequest) returns (BillingResponse);
}

message BillingRequest {
  string patientId = 1;
  string name = 2;
  string email = 3;
}

message BillingResponse {
  string accountId = 1;
  string status = 2;
}
```

**Proto3 Features:**
- **Service Definition**: Single RPC method for billing account creation
- **Message Types**: Request and response contracts
- **Field Numbers**: Unique identifiers for each field
- **Java Package**: Generated classes package

---

## Inter-Service Communication

### gRPC Client Implementation

```java
@Service
public class BillingServiceGrpcClient {

  private static final Logger log = LoggerFactory.getLogger(
      BillingServiceGrpcClient.class);
  private final BillingServiceGrpc.BillingServiceBlockingStub blockingStub;

  public BillingServiceGrpcClient(
      @Value("${billing.service.address:localhost}") String serverAddress,
      @Value("${billing.service.grpc.port:9101}") int serverPort) {

    log.info("Connecting to Billing Service GRPC service at {}:{}",
        serverAddress, serverPort);

    ManagedChannel channel = ManagedChannelBuilder.forAddress(serverAddress,
        serverPort).usePlaintext().build();

    blockingStub = BillingServiceGrpc.newBlockingStub(channel);
  }

  public BillingResponse createBillingAccount(String patientId, String name,
      String email) {

    BillingRequest request = BillingRequest.newBuilder().setPatientId(patientId)
        .setName(name).setEmail(email).build();

    BillingResponse response = blockingStub.createBillingAccount(request);
    log.info("Received response from billing service via GRPC: {}", response);
    return response;
  }
}
```

**gRPC Client Features:**
- **Configuration Injection**: Server address and port from properties
- **Blocking Stub**: Synchronous communication
- **Plaintext Channel**: No TLS for development
- **Request Building**: Protocol buffer message construction
- **Response Logging**: Debug information for troubleshooting

### Communication Flow

1. **Patient Creation Request** → Patient Service REST API
2. **Validation** → Input validation and business rules
3. **Database Save** → Patient record persistence
4. **gRPC Call** → Billing Service communication
5. **Billing Account Creation** → Billing Service business logic
6. **Response** → Billing account details
7. **API Response** → Patient data with billing confirmation

---

## Data Flow & Architecture Patterns

### Layered Architecture Pattern

```
┌─────────────────┐
│   Controller    │ ← REST API Layer
├─────────────────┤
│    Service      │ ← Business Logic Layer
├─────────────────┤
│   Repository    │ ← Data Access Layer
├─────────────────┤
│    Database     │ ← Persistence Layer
└─────────────────┘
```

### CQRS Pattern Elements

**Command Operations (Write):**
- `createPatient()` - Creates new patient and billing account
- `updatePatient()` - Updates existing patient
- `deletePatient()` - Removes patient

**Query Operations (Read):**
- `getPatients()` - Retrieves all patients

### Saga Pattern (Distributed Transaction)

```
Patient Creation Saga:
1. Validate patient data
2. Save patient to database
3. Call billing service (gRPC)
4. If billing fails → Rollback patient creation
5. If successful → Complete transaction
```

### DTO Pattern

**Request DTOs:**
- Input validation
- API contract definition
- Separation of concerns

**Response DTOs:**
- Controlled data exposure
- API versioning support
- Performance optimization

### Repository Pattern

**Benefits:**
- Abstraction of data access
- Testability
- Centralized query logic
- Type safety

### Exception Handling Pattern

**Global Exception Handler:**
- Centralized error handling
- Consistent error responses
- Logging strategy
- HTTP status code mapping

---

## Configuration Management

### Patient Service Configuration

```properties
# Application
spring.application.name=patient-service

# PostgreSQL Database
spring.datasource.url=jdbc:postgresql://patient-service-db:5432/db
spring.datasource.username=admin_user
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA/Hibernate
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.sql.init.mode=always

# Server
server.port=4000
logging.level.root=info

# Billing Service
billing.service.address=biiling-service
billing.service.grpc.port=9001
```

### Billing Service Configuration

```properties
# Application
spring.application.name=billing-service

# Server
server.port=4001
grpc.server.port=9001
```

### Environment Variables Strategy

**Development:**
- Local PostgreSQL instance
- Direct service communication
- Debug logging enabled

**Production:**
- Docker container networking
- Externalized configuration
- Structured logging

---

## Deployment & Docker

### Multi-Stage Docker Builds

**Patient Service Dockerfile:**
```dockerfile
# Build stage
FROM maven:3.9.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src
RUN mvn clean package -DskipTests

# Run stage
FROM eclipse-temurin:21-jdk AS runner
WORKDIR /app
COPY --from=builder /app/target/patient-service-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 4000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Benefits:**
- **Smaller Images**: Only runtime dependencies in final image
- **Security**: No build tools in production image
- **Performance**: Faster startup and smaller attack surface

### Service Dependencies

**Patient Service:**
- **Port**: 4000 (HTTP REST API)
- **Database**: PostgreSQL on port 5432
- **Dependencies**: Billing Service (gRPC port 9001)

**Billing Service:**
- **HTTP Port**: 4101 (Spring Boot admin)
- **gRPC Port**: 9001 (Service communication)

### Docker Compose Setup (Implied)

```yaml
services:
  patient-service:
    build: ./patient-service
    ports:
      - "4000:4000"
    depends_on:
      - patient-service-db
      - billing-service
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db

  billing-service:
    build: ./billing-service
    ports:
      - "4101:4101"
      - "9001:9001"

  patient-service-db:
    image: postgres:latest
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=admin_user
      - POSTGRES_PASSWORD=password
```

---

## API Examples & Testing

### REST API Examples

**Create Patient:**
```http
POST http://localhost:4000/patients
Content-Type: application/json

{
  "name": "Vedansh Mahheshwari",
  "email": "ved@gmail.com",
  "address": "bengaluru",
  "dateOfBirth": "1995-09-09",
  "registeredDate": "2024-11-28"
}
```

**Get All Patients:**
```http
GET http://localhost:4000/patients
```

**Update Patient:**
```http
PUT http://localhost:4000/patients/{uuid}
Content-Type: application/json

{
  "name": "Updated Name",
  "email": "updated@email.com",
  "address": "Updated Address",
  "dateOfBirth": "1995-09-09"
}
```

**Delete Patient:**
```http
DELETE http://localhost:4000/patients/{uuid}
```

### gRPC Testing

**Create Billing Account:**
```grpc
GRPC localhost:9001/BillingService/CreateBillingAccount

{
  "patientId": "12333",
  "name": "John Doe",
  "email": "john.doe@example.com"
}
```

**Expected Response:**
```json
{
  "accountId": "12345",
  "status": "ACTIVE"
}
```

### Error Responses

**Validation Error:**
```json
{
  "name": "Name is required",
  "email": "Email should be valid"
}
```

**Business Rule Violation:**
```json
{
  "message": "Email address already exists"
}
```

---

## Development Workflow

### 1. Local Development Setup

```bash
# Start PostgreSQL
docker run -d --name postgres -p 5432:5432 -e POSTGRES_DB=db -e POSTGRES_USER=admin_user -e POSTGRES_PASSWORD=password postgres:latest

# Build and run billing service
cd billing-service
mvn clean package
java -jar target/billing-service-0.0.1-SNAPSHOT.jar

# Build and run patient service
cd ../patient-service
mvn clean package
java -jar target/patient-service-0.0.1-SNAPSHOT.jar
```

### 2. Testing Strategy

**Unit Tests:**
- Service layer business logic
- Mapper classes
- Validation rules

**Integration Tests:**
- Repository layer with test database
- Controller layer with MockMvc
- gRPC client with mocked server

**API Tests:**
- REST API endpoints
- gRPC service methods
- Error scenarios

### 3. Code Generation

**Protocol Buffers:**
```bash
# Maven plugin generates Java classes from .proto files
mvn clean compile
```

**Generated Classes:**
- `BillingServiceGrpc.java` - gRPC service stubs
- `BillingRequest.java` - Request message
- `BillingResponse.java` - Response message

### 4. Debugging

**Application Logs:**
- Spring Boot actuator endpoints
- Custom logging in service methods
- gRPC request/response logging

**Database Debugging:**
- H2 console (for development)
- PostgreSQL logs
- Hibernate SQL logging

### 5. Monitoring

**Health Checks:**
- Spring Boot Actuator `/health` endpoint
- Database connectivity checks
- gRPC service availability

**Metrics:**
- Request/response times
- Error rates
- Database query performance

---

## Key Design Decisions & Best Practices

### 1. **Microservices Boundaries**
- **Patient Service**: Owns patient lifecycle and data
- **Billing Service**: Owns billing account lifecycle
- **Clear Separation**: Each service has single responsibility

### 2. **API Design**
- **RESTful**: Standard HTTP methods and status codes
- **OpenAPI**: Self-documenting APIs with Swagger UI
- **Validation**: Comprehensive input validation with groups

### 3. **Data Consistency**
- **Saga Pattern**: Distributed transaction coordination
- **Compensating Actions**: Rollback strategies for failures
- **Idempotency**: Safe retry operations

### 4. **Error Handling**
- **Custom Exceptions**: Business-specific error types
- **Global Handler**: Centralized error processing
- **Consistent Responses**: Standardized error format

### 5. **Performance Considerations**
- **Lazy Loading**: JPA relationships optimization
- **DTO Pattern**: Prevents over-fetching
- **Connection Pooling**: Database connection management

### 6. **Security Considerations**
- **Input Validation**: Prevents injection attacks
- **Data Sanitization**: Safe data handling
- **Access Control**: Future authentication/authorization hooks

---

## Future Enhancements

### 1. **Service Mesh**
- **Istio/Service Mesh**: Advanced inter-service communication
- **Circuit Breakers**: Fault tolerance
- **Service Discovery**: Dynamic service location

### 2. **Event-Driven Architecture**
- **Apache Kafka**: Asynchronous event processing
- **Event Sourcing**: Audit trail and state reconstruction
- **CQRS**: Separate read/write models

### 3. **Observability**
- **Distributed Tracing**: Request flow visualization
- **Metrics Collection**: Prometheus/Grafana integration
- **Log Aggregation**: ELK stack implementation

### 4. **Security**
- **OAuth2/JWT**: Authentication and authorization
- **API Gateway**: Centralized access control
- **Data Encryption**: Sensitive data protection

### 5. **Database Optimization**
- **Read Replicas**: Performance scaling
- **Caching Layer**: Redis for frequently accessed data
- **Database Sharding**: Horizontal scaling strategy

---

This comprehensive guide covers the complete backend architecture of your Java Spring Boot and gRPC-based patient management system. The system demonstrates modern microservices patterns, clean architecture principles, and production-ready practices.