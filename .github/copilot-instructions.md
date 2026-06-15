# Copilot Instructions — Hospital Management System (HMS)

## Build & Test Commands

```bash
# Build
mvn clean install

# Run all tests
mvn test

# Run a single test class
mvn test -Dtest=AppointmentServiceTest

# Run a single test method
mvn test -Dtest=AppointmentServiceTest#bookAppointment_Success

# Run the application
mvn spring-boot:run
```

The app starts on `http://localhost:8080/api/v1`. Swagger UI is at `/api/v1/swagger-ui.html`.

Requires a running MySQL instance at `localhost:3306` with database `hms_db` (auto-created on first run). Default credentials: `root / root` (see `application.yml`).

---

## Architecture

Spring Boot 3.2.3 / Java 17 REST API with a **feature-based package layout** under `com.hms`. Each feature module (`appointment`, `auth`, `billing`, `doctor`, `medicalrecord`, `patient`, `prescription`) is self-contained with these layers:

```
<feature>/
  controller/     # REST controller
  dto/            # Request/Response DTOs
  entity/         # JPA entity
  mapper/         # MapStruct mapper interface
  repository/     # Spring Data JPA repository
  service/        # Interface + impl/ subdirectory
```

Cross-cutting concerns live in dedicated packages:
- `com.hms.common` — `BaseEntity`, `ApiResponse<T>`, shared enums
- `com.hms.config` — `SecurityConfig`, `SwaggerConfig`, `AuditConfig`
- `com.hms.exception` — custom exceptions + `GlobalExceptionHandler`
- `com.hms.security` — JWT filter, token provider, `UserPrincipal`

**Authentication flow**: JWT is verified in `JwtAuthenticationFilter` → `CustomUserDetailsService` loads the `User` entity → Spring Security `@EnableMethodSecurity` handles method-level authorization in addition to URL rules in `SecurityConfig`.

**Domain relationships**: `Appointment` links `Patient` ↔ `Doctor`. `Prescription` and `MedicalRecord` also reference both. `Billing` is standalone (created per patient visit).

---

## Key Conventions

### Entities
- All entities extend `BaseEntity`, which auto-populates `createdAt`, `updatedAt`, `createdBy`, `updatedBy` via Spring Data Auditing.
- Every entity uses Lombok (`@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`).
- Enums are persisted as `EnumType.STRING`.

### Business codes
Each domain object has a human-readable code generated in the service layer:
| Entity | Code format | Example |
|---|---|---|
| Patient | `PAT-YYYYMM-NNNN` | `PAT-202601-0001` |
| Doctor | `DOC-SPEC-YYYY-NNNN` | `DOC-CAR-2026-0001` |
| Appointment | `APT-YYYYMMDD-NNNNN` | `APT-20260615-00001` |
| Prescription | `RX-YYYYMMDD-NNNNN` | similar pattern |

### DTOs & Responses
- Controllers always return `ApiResponse<T>` (from `com.hms.common.dto`). Use the static factory methods: `ApiResponse.success(data)`, `ApiResponse.success(message, data)`, `ApiResponse.error(message)`.
- MapStruct mappers are configured with `defaultComponentModel=spring` (injected as Spring beans). Mapper interfaces live in `<feature>/mapper/`.

### Exceptions
Throw domain-specific exceptions from services; do **not** return error responses directly. The `GlobalExceptionHandler` maps them to HTTP responses:
| Exception | HTTP status |
|---|---|
| `ResourceNotFoundException` | 404 |
| `AppointmentConflictException` | 409 |
| `DoctorUnavailableException` | 422 |
| `BusinessValidationException` | 400 |

### Transactions
- All write service methods are `@Transactional`.
- All read service methods are `@Transactional(readOnly = true)`.

### Security / Roles
Roles: `ADMIN`, `DOCTOR`, `RECEPTIONIST`, `PATIENT`. Role-based URL rules are defined in `SecurityConfig`. Public endpoints: `/auth/**`, Swagger paths, `/actuator/**`.

### Tests
- Unit tests use `@ExtendWith(MockitoExtension.class)` with `@Mock` / `@InjectMocks` — no Spring context.
- Assertions use AssertJ (`assertThat`, `assertThatThrownBy`).
- Test classes mirror the `impl` class under `test/java/com/hms/<feature>/service/`.

### MapStruct + Lombok
All `toEntity()` mapper methods must include `@BeanMapping(builder = @Builder(disableBuilder = true))` to force MapStruct to use setters instead of the builder. Omitting this causes a compile error because Lombok's `@Builder` does not include fields inherited from `BaseEntity`.

---

## GHAS Vulnerability Management

### Multi-Agent Orchestration
A two-workflow, multi-agent system lives in `.github/agents/` for automated Dependabot vulnerability remediation.

```
.github/
  agents/
    dependabot-vuln-orchestrator.md   ← entry point (@dependabot-vuln-orchestrator)
    w1-fetcher.md                     ← runs fetch script, produces Excel
    w1-sorter.md                      ← sorts Excel by service + severity
    w1-jira-manager.md                ← Jira dedup check + ticket creation
    w2-context-builder.md             ← fetches alerts + parses pom.xml
    w2-fixer.md                       ← patches pom.xml (CRITICAL first)
    w2-validator.md                   ← mvn compile + test + smoke check
    w2-reporter.md                    ← raises PR + updates Jira to In Review
  scripts/
    fetch_dependabot_alerts.py        ← GitHub REST API → color-coded Excel
```

**Invoke via Copilot Chat:**
```
@dependabot-vuln-orchestrator Run both workflows for HMS
@dependabot-vuln-orchestrator Run ingest only
@dependabot-vuln-orchestrator Resolve HMS with Jira ticket SEC-101
```

### Workflow 1 — Alert Ingestion
1. `@w1-fetcher` runs `fetch_dependabot_alerts.py` (requires `GITHUB_TOKEN` env var)
2. `@w1-sorter` sorts Excel by service name, then CRITICAL → HIGH → MEDIUM → LOW
3. `@w1-jira-manager` searches Jira (Backlog + In Dev) by CVE label + service label — skips if found, creates if not

**Excel columns:** Service | Repo | Alert # | Severity | CVE ID | Package | Vulnerable Range | Safe Version | Manifest | Scope | Summary | Alert URL | **Jira Key** | **Jira Status**

### Workflow 2 — Vulnerability Resolver
Fix strategy rules enforced by `@w2-fixer`:
- **Property-backed versions** (`${some.version}`) → update `<properties>` block only — one change covers all usages (preferred)
- **Inline versions** → update `<version>` tag directly
- **BOM-managed** (no `<version>` tag) → skip, note in PR
- Sibling groups (`jjwt-*`, `log4j-*`, `jackson-*`) must always share the same version — when fixing one, update all siblings

Validation order in `@w2-validator`: `mvn dependency:tree` → `mvn compile` → `mvn test` → `spring-boot:run` health check. Individual failing fixes are reverted, not the whole file.

### Dependabot Schedule
Configured in `.github/dependabot.yml` — weekly on Mondays at 09:00 IST, maven ecosystem, max 5 open PRs.
