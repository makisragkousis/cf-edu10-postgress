
Code

## Quick Start

### Prerequisites

- Java 21+
- Docker & Docker Compose
- Gradle (or use the included wrapper: `./gradlew` on Linux/Mac, `gradlew.bat` on Windows)

### 1. Start PostgreSQL

```bash
docker-compose up -d
This starts a PostgreSQL 18 container with credentials from .env:

Host: localhost
Port: 5433
Database: edu10restdb
User: edu10user
Password: C0d1ngF@
2. Build the Application
bash
./gradlew build          # Linux/Mac
gradlew.bat build        # Windows
3. Run the Application
bash
./gradlew bootRun        # Linux/Mac
gradlew.bat bootRun      # Windows
The application starts on http://localhost:8080 with the dev Spring profile.

4. Access API Documentation
Once running, open your browser:

Code
http://localhost:8080/swagger-ui/index.html
Swagger UI does not require authentication and displays all available endpoints.

Configuration
Configuration is split by Spring profile, with secrets loaded from .env:

.env File
Located at the repo root, contains:

Dotenv
POSTGRES_HOST=localhost
POSTGRES_PORT=5433
POSTGRES_DB=edu10restdb
POSTGRES_USER=edu10user
POSTGRES_PASSWORD=C0d1ngF@
JWT_SECRET_KEY=QAvIUJMhYX0NeW2BvjuzaiU23MXAOHX5OGNwLFiPmJo3m7qNz92kFjYBVQwDo0sk
These are imported in application-dev.properties via:

properties
spring.config.import=optional:file:.env[.properties]
⚠️ Security Note: Never commit .env with real credentials to version control. Add .env to .gitignore and share credentials securely with your team.

Database Migrations
Flyway manages database schema through SQL migration files in src/main/resources/db/migration/:

V1__initial_schema.sql — Creates main tables (teachers, users, personal_info, attachments, etc.)
V2__insert_regions.sql — Seeds reference data (Greek regions)
V3__insert_roles_capabilites.sql — Creates roles (ADMIN, EMPLOYEE, TEACHER) and capabilities
Migrations run automatically on application startup. Reference queries are stored in src/main/resources/db/queries/ for documentation and debugging.

Authentication & Authorization
JWT Flow
Login: POST /api/auth/login with credentials
Token Issue: Server issues a JWT (HS256) with user roles/capabilities as claims
Protected Requests: Include Authorization: Bearer <token> header
Validation: JwtAuthenticationFilter validates the token and populates SecurityContext
Roles & Capabilities
Roles are linked to capabilities via a many-to-many relationship:

ADMIN — Full system access
EMPLOYEE — Internal staff access (VIEW_TEACHERS, CREATE_TEACHER, etc.)
TEACHER — Limited self-service access (VIEW_ONLY_TEACHER for own data)
Authorization is enforced at the service layer via @PreAuthorize annotations:

Java
@PreAuthorize("hasAuthority('VIEW_TEACHERS')")
public Page<TeacherReadOnlyDTO> getTeachers(...) { ... }
API Endpoints Overview
Teachers
GET /api/teachers — List teachers (with filtering & pagination)
GET /api/teachers/{uuid} — Get a single teacher
POST /api/teachers — Create a teacher (requires nested user & personal info)
PUT /api/teachers/{uuid} — Update teacher
DELETE /api/teachers/{uuid} — Soft-delete a teacher
Users
GET /api/users — List users
GET /api/users/{uuid} — Get a single user
POST /api/users — Create a user
PUT /api/users/{uuid} — Update user
DELETE /api/users/{uuid} — Soft-delete a user
Authentication
POST /api/auth/login — Authenticate and get JWT
File Uploads
POST /api/teachers/{uuid}/upload-amka — Upload AMKA document (multipart/form-data)
See Swagger UI at /swagger-ui/index.html for detailed request/response schemas.

Data Model
Key Entities
Teacher

UUID-based identifier (external API reference)
Linked to exactly one User and one PersonalInfo
Supports soft-delete via deletedAt field
Audit timestamps: createdAt, updatedAt
User

Credentials & authentication
Many-to-many link to roles (via role_capabilities)
Soft-delete support
PersonalInfo

Name, email, phone, address, region
Linked to a single teacher
Attachment

File upload metadata (filename, MIME type, size, path)
Linked to a teacher
Role

ADMIN, EMPLOYEE, TEACHER
Many-to-many link to capabilities
IDs & References
External APIs should always reference entities by uuid (not internal id)
Internal Long id is used for JPA primary key
UUIDs are auto-generated on entity creation
File Uploads
Teachers can upload AMKA (Social Security) documents:

bash
curl -X POST \
  -F "file=@amka.pdf" \
  http://localhost:8080/api/teachers/{uuid}/upload-amka \
  -H "Authorization: Bearer <token>"
Behavior:

File is stored to file.upload.dir (default: uploads/)
MIME type is detected via Apache Tika
Metadata is persisted as an Attachment entity
Invalid file types are rejected
Testing
Run All Tests
bash
./gradlew test              # Linux/Mac
gradlew.bat test            # Windows
Run a Single Test Class
bash
./gradlew test --tests "gr.aueb.cf.edu10restapppostgress.Edu10RestappPostgressApplicationTests"
Run a Single Test Method
bash
./gradlew test --tests "FullyQualifiedClassName.methodName"
Test Setup
Tests use Testcontainers with the PostgreSQL module to spin up a real database for integration tests:

Docker must be running and accessible
Each test gets a fresh, isolated database
Migrations run automatically within the test container
Logging
The application uses structured logging with MDC (Mapped Diagnostic Context):

MDCLoggingFilter adds request-scoped context (e.g., request ID) to the MDC
Useful for tracing requests through logs in production environments
Error Handling
Exception Hierarchy
EntityNotFoundException — Requested entity not found
EntityAlreadyExistsException — Duplicate or constraint violation
EntityInvalidArgumentException — Invalid input data
ValidationException — Bean validation errors
FileUploadException — File upload validation failure
Error Response Format
JSON
{
  "status": 400,
  "message": "Validation failed",
  "timestamp": "2026-09-15T12:34:56Z",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email address"
    }
  ]
}
See core/ErrorHandler for details on global exception handling.

Notes
Stale Copy
A duplicate, stale copy of several classes exists in ./eduapp/ at the repo root (outside src/). This is not part of the Gradle build — the canonical source is src/main/java/gr/aueb/cf/eduapp/. The duplicate should be removed in a future cleanup.

Package Name Mismatch
Application package: gr.aueb.cf.eduapp
Test default package: gr.aueb.cf.edu10restapppostgress
This is a leftover from project generation and has no functional impact.

Development Workflow
Create/modify entities in model/
Create Flyway migration in src/main/resources/db/migration/ if schema changes
Create repository in repository/ (extend JpaRepository or add custom queries)
Implement service in service/ with business logic and @PreAuthorize checks
Create DTOs in dto/ for request/response
Add controller in api/ with @RestController and endpoint methods
Add tests for the service/controller logic
Run ./gradlew test to verify
Commit and push
Contributing
Fork the repository
Create a feature branch (git checkout -b feature/your-feature)
Make changes and write tests
Run ./gradlew build and ./gradlew test locally
Commit with clear messages
Push and open a pull request