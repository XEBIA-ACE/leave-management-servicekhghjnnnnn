# AGENTS.md — Leave Management Service

## 1. Stack

| Technology | Role |
|---|---|
| **Java 21 (LTS)** | Primary language; use records, sealed classes, and virtual threads where appropriate |
| **Spring Boot 3.x** | Application framework; web, data, messaging, validation, actuator starters |
| **Spring Data JPA / Hibernate** | ORM layer for PostgreSQL persistence |
| **Spring AMQP (RabbitMQ)** | Async messaging for leave approval events and calendar integration notifications |
| **PostgreSQL 16** | Primary relational data store |
| **Flyway** | Database schema versioning and migrations |
| **MapStruct** | Compile-time DTO ↔ Entity mapping |
| **Lombok** | Boilerplate reduction (builders, loggers, constructors) |
| **SpringDoc OpenAPI 3** | Auto-generated API documentation at `/api-docs` |
| **Spring Security** | JWT-based authentication and role-based access control |
| **Resilience4j** | Circuit breaker and retry for external calendar system calls |
| **Testcontainers** | Integration tests with real PostgreSQL and RabbitMQ containers |
| **JUnit 5 + Mockito** | Unit testing framework |
| **JaCoCo** | Code coverage enforcement |
| **Docker / Docker Compose** | Containerisation and local development environment |
| **GitHub Actions** | CI pipeline |

---

## 2. Project Structure

```
leave-management-service/
├── AGENTS.md                          # This file
├── tasks.md                           # Agent-generated task tracker (created before coding begins)
├── README.md                          # Human-readable service overview
├── pom.xml                            # Maven build descriptor; all dependency versions pinned
├── .env.example                       # Environment variable template (never commit .env)
├── .gitignore
├── Dockerfile                         # Multi-stage production image
├── docker-compose.yml                 # Local dev: app + postgres + rabbitmq
├── docker-compose.test.yml            # Integration test overrides
│
├── .github/
│   └── workflows/
│       └── ci.yml                     # GitHub Actions pipeline
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/leave/
│   │   │       ├── LeaveManagementApplication.java   # @SpringBootApplication entry point
│   │   │       │
│   │   │       ├── config/
│   │   │       │   ├── RabbitMQConfig.java           # Exchange, queue, binding bean definitions
│   │   │       │   ├── SecurityConfig.java            # JWT filter chain, role mappings
│   │   │       │   ├── JpaConfig.java                 # Auditing, transaction manager settings
│   │   │       │   ├── Resilience4jConfig.java        # Circuit breaker / retry beans
│   │   │       │   └── OpenApiConfig.java             # SpringDoc customisation
│   │   │       │
│   │   │       ├── domain/
│   │   │       │   ├── model/
│   │   │       │   │   ├── LeaveApplication.java      # JPA entity
│   │   │       │   │   ├── LeaveType.java             # JPA entity
│   │   │       │   │   ├── LeaveBalance.java          # JPA entity
│   │   │       │   │   └── enums/
│   │   │       │   │       ├── LeaveStatus.java       # PENDING, APPROVED, REJECTED, CANCELLED
│   │   │       │   │       └── LeaveCategory.java     # ANNUAL, SICK, MATERNITY, etc.
│   │   │       │   │
│   │   │       │   └── event/
│   │   │       │       ├── LeaveApprovedEvent.java    # Domain event record
│   │   │       │       ├── LeaveRejectedEvent.java    # Domain event record
│   │   │       │       └── LeaveSubmittedEvent.java   # Domain event record
│   │   │       │
│   │   │       ├── repository/
│   │   │       │   ├── LeaveApplicationRepository.java  # Spring Data JPA interface
│   │   │       │   ├── LeaveTypeRepository.java
│   │   │       │   └── LeaveBalanceRepository.java
│   │   │       │
│   │   │       ├── service/
│   │   │       │   ├── LeaveApplicationService.java   # Core business logic interface
│   │   │       │   ├── LeaveTypeService.java
│   │   │       │   ├── LeaveBalanceService.java
│   │   │       │   ├── LeaveApprovalService.java      # Approval / rejection workflow
│   │   │       │   └── impl/
│   │   │       │       ├── LeaveApplicationServiceImpl.java
│   │   │       │       ├── LeaveTypeServiceImpl.java
│   │   │       │       ├── LeaveBalanceServiceImpl.java
│   │   │       │       └── LeaveApprovalServiceImpl.java
│   │   │       │
│   │   │       ├── messaging/
│   │   │       │   ├── publisher/
│   │   │       │   │   └── LeaveEventPublisher.java   # Publishes domain events to RabbitMQ
│   │   │       │   └── consumer/
│   │   │       │       └── CalendarEventConsumer.java # Consumes calendar ack/nack messages
│   │   │       │
│   │   │       ├── integration/
│   │   │       │   └── calendar/
│   │   │       │       ├── CalendarClient.java        # Interface for calendar system calls
│   │   │       │       ├── CalendarClientImpl.java    # RestClient impl with Resilience4j
│   │   │       │       └── dto/
│   │   │       │           ├── CalendarEventRequest.java
│   │   │       │           └── CalendarEventResponse.java
│   │   │       │
│   │   │       ├── web/
│   │   │       │   ├── controller/
│   │   │       │   │   ├── LeaveApplicationController.java  # /api/v1/leave-applications
│   │   │       │   │   ├── LeaveTypeController.java         # /api/v1/leave-types
│   │   │       │   │   ├── LeaveBalanceController.java      # /api/v1/leave-balances
│   │   │       │   │   └── LeaveApprovalController.java     # /api/v1/leave-applications/{id}/approve|reject
│   │   │       │   │
│   │   │       │   ├── dto/
│   │   │       │   │   ├── request/
│   │   │       │   │   │   ├── LeaveApplicationRequest.java
│   │   │       │   │   │   ├── LeaveTypeRequest.java
│   │   │       │   │   │   └── LeaveApprovalRequest.java
│   │   │       │   │   └── response/
│   │   │       │   │       ├── LeaveApplicationResponse.java
│   │   │       │   │       ├── LeaveTypeResponse.java
│   │   │       │   │       ├── LeaveBalanceResponse.java
│   │   │       │   │       └── PagedResponse.java
│   │   │       │   │
│   │   │       │   └── mapper/
│   │   │       │       ├── LeaveApplicationMapper.java  # MapStruct interface
│   │   │       │       ├── LeaveTypeMapper.java
│   │   │       │       └── LeaveBalanceMapper.java
│   │   │       │
│   │   │       └── exception/
│   │   │           ├── GlobalExceptionHandler.java    # @RestControllerAdvice
│   │   │           ├── LeaveNotFoundException.java
│   │   │           ├── InsufficientLeaveBalanceException.java
│   │   │           ├── InvalidLeaveStatusTransitionException.java
│   │   │           └── ErrorResponse.java             # Standardised error payload record
│   │   │
│   │   └── resources/
│   │       ├── application.yml                        # Base configuration
│   │       ├── application-local.yml                  # Local dev overrides
│   │       ├── application-test.yml                   # Test profile (Testcontainers)
│   │       └── db/migration/
│   │           ├── V1__create_leave_types.sql
│   │           ├── V2__create_leave_applications.sql
│   │           ├── V3__create_leave_balances.sql
│   │           └── V4__seed_default_leave_types.sql
│   │
│   └── test/
│       └── java/
│           └── com/company/leave/
│               ├── unit/
│               │   ├── service/
│               │   │   ├── LeaveApplicationServiceTest.java
│               │   │   ├── LeaveApprovalServiceTest.java
│               │   │   └── LeaveBalanceServiceTest.java
│               │   ├── messaging/
│               │   │   └── LeaveEventPublisherTest.java
│               │   └── web/
│               │       ├── LeaveApplicationControllerTest.java  # @WebMvcTest
│               │       └── LeaveApprovalControllerTest.java
│               │
│               └── integration/
│                   ├── AbstractIntegrationTest.java    # Testcontainers base class
│                   ├── LeaveApplicationIT.java         # Full slice integration tests
│                   ├── LeaveApprovalIT.java
│                   └── MessagingIT.java                # RabbitMQ publish/consume round-trip
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder.

### Step 1 — Read All Specifications
- Parse every provided specification document thoroughly before writing any code.
- Identify all entities, status transitions, API endpoints, messaging contracts, and external integrations.
- Flag any ambiguities in `tasks.md` under a `## Clarifications Needed` section.

### Step 2 — Create `tasks.md`
Create `tasks.md` in the project root before writing any source code. Structure it as follows:

```markdown
# tasks.md

## Status Legend
- [ ] Not started
- [~] In progress
- [x] Complete

## Phase 1: Project Scaffolding
- [ ] Initialise Maven project with all required dependencies
- [ ] Configure application.yml profiles (base, local, test)
- [ ] Set up Flyway migration scripts
- [ ] Configure Docker and docker-compose files

## Phase 2: Domain Layer
- [ ] Define JPA entities with proper constraints and auditing
- [ ] Define enums for LeaveStatus and LeaveCategory
- [ ] Define domain event records

## Phase 3: Repository Layer
- [ ] Implement Spring Data JPA repositories with custom queries

## Phase 4: Service Layer
- [ ] Implement leave application CRUD
- [ ] Implement approval/rejection workflow with status transition validation
- [ ] Implement leave balance deduction and restoration logic
- [ ] Implement calendar integration service calls

## Phase 5: Messaging Layer
- [ ] Configure RabbitMQ exchanges, queues, and bindings
- [ ] Implement LeaveEventPublisher
- [ ] Implement CalendarEventConsumer

## Phase 6: Web Layer
- [ ] Implement all REST controllers
- [ ] Implement request/response DTOs with Bean Validation annotations
- [ ] Implement MapStruct mappers
- [ ] Implement GlobalExceptionHandler

## Phase 7: Security
- [ ] Configure JWT authentication filter
- [ ] Apply role-based access: EMPLOYEE, MANAGER, HR_ADMIN

## Phase 8: Testing
- [ ] Write unit tests for all service classes (target ≥ 90% coverage)
- [ ] Write @WebMvcTest controller tests
- [ ] Write Testcontainers integration tests
- [ ] Run JaCoCo report and verify thresholds pass

## Phase 9: Docker & CI
- [ ] Validate Dockerfile builds successfully
- [ ] Validate docker-compose brings up full stack
- [ ] Validate CI pipeline passes all stages

## Clarifications Needed
- (list any here)
```

Update task statuses as work progresses.

### Step 3 — Implement in Phase Order
- Complete each phase fully before starting the next.
- After each phase, run the relevant tests and confirm they pass.
- Commit logically scoped changes per phase (message: `feat(phase-N): <description>`).

### Step 4 — Test
- Run `./mvnw verify` after every phase.
- Run `./mvnw jacoco:report` and confirm the coverage report shows ≥ 90% line and branch coverage on `src/main`.
- Fix any failures before advancing.

### Step 5 — Validate
- Start the full stack with `docker-compose up --build`.
- Confirm `/actuator/health` returns `{"status":"UP"}`.
- Confirm `/api-docs` returns a valid OpenAPI document.
- Confirm RabbitMQ management UI shows expected exchanges and queues.
- Mark all `tasks.md` items `[x]` and append a `## Completion Summary` section.

---

## 4. Coding Conventions

### Naming
| Artifact | Convention | Example |
|---|---|---|
| Packages | `com.company.leave.<layer>` | `com.company.leave.service` |
| Entity classes | PascalCase noun | `LeaveApplication` |
| Service interfaces | PascalCase + `Service` suffix | `LeaveApprovalService` |
| Service impls | Interface name + `Impl` | `LeaveApprovalServiceImpl` |
| Controllers | Resource noun + `Controller` | `LeaveApplicationController` |
| DTOs | Noun + `Request` / `Response` | `LeaveApplicationRequest` |
| Mappers | Entity noun + `Mapper` | `LeaveApplicationMapper` |
| RabbitMQ queues | `kebab-case` | `leave.approved.queue` |
| RabbitMQ exchanges | `kebab-case` with `.exchange` suffix | `leave-events.exchange` |
| Database tables | `snake_case` plural | `leave_applications` |
| Flyway scripts | `V{n}__{description}.sql` | `V2__create_leave_applications.sql` |
| Constants | `UPPER_SNAKE_CASE` in dedicated `*Constants` class | `LeaveConstants.MAX_ANNUAL_DAYS` |

### Architecture Patterns
- **Layered architecture**: `web → service → repository`. Controllers must not call repositories directly.
- **Interface-first services**: All service classes implement an interface; inject via interface type.
- **Domain events**: Publish events via `LeaveEventPublisher` after successful DB commits (use `@TransactionalEventListener(phase = AFTER_COMMIT)`).
- **Status machine**: Enforce `LeaveStatus` transitions in `LeaveApprovalServiceImpl`. Throw `InvalidLeaveStatusTransitionException` for illegal transitions. Valid transitions:
  - `PENDING → APPROVED`
  - `PENDING → REJECTED`
  - `PENDING → CANCELLED`
  - `APPROVED → CANCELLED` (within cancellation window only)
- **DTO boundary**: Entities must never be returned from controllers. Always map to response DTOs via MapStruct.
- **Validation**: Use Bean Validation (`@NotNull`, `@NotBlank`, `@FutureOrPresent`, `@Min`) on all request DTOs. Validate at the controller layer with `@Valid`.
- **Pagination**: All list endpoints must accept `Pageable` and return `PagedResponse<T>`.
- **Auditing**: All entities extend a `BaseAuditEntity` with `createdAt`, `updatedAt`, `createdBy`, `updatedBy` using Spring Data JPA auditing (`@EnableJpaAuditing`).
- **Logging**: Use `@Slf4j` (Lombok). Log at `INFO