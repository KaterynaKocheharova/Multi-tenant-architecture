# 🧱 C4 Container Diagram

The system is composed of three main containers:

---

## 🌐 Web Application (React)

### Responsibilities

- Show tables of students, teachers, tenants, events with filtration
- Forms for events creations and registrations as well as lesson plans creations
- UI for prefilled reports generations with ability to edit them.
- Student, teacher profiles UI.
- Role-based UI rendering (RBAC)
- Communicates with backend API via HTTP/JSON

### Reason for choice

- Highly interactive, form-driven system
- Strong component-based architecture fits domain modules
- Efficient state management for dashboards and workflows
- Works naturally with TypeScript shared with backend

---

## ⚙️ Backend API (Node.js)

### Responsibilities

- Core business logic for all bounded contexts
- Authentication, authorization, tenant isolation
- Tenant lifecycle management (school registration, provisioning, activation)
- Exposes REST API to frontend

### Main Internal Modules

- AuthModule
- EventsModule
- TeachersModule / StudentsModule
- LessonPlansModule
- ReportsModule
- TenantManagementModule

### Architecture style

- Modular monolith (DDD-oriented)
- CQRS-lite separation (Command / Query services)
- Strategy pattern for event/report type variations

### Reason for choice

- Non-blocking I/O for high concurrency workloads with ability to use additional threads when needed for tasks with complext calculations like report generations
- Efficient handling of many simultaneous user actions
- Strong ecosystem for modular backend design
- Natural fit with TypeScript full-stack development

---

## 🗄️ Database (PostgreSQL)

### Responsibilities

- Stores all persistent application data
- Enforces relational integrity between:
  - Schools (tenants)
  - Users
  - Events
  - Lesson plans, etc.
- Supports complex queries for reporting and aggregation

### Reason for choice

- Strong relational model fits highly connected domain
- ACID transactions ensure consistency in grading and attendance
- Efficient joins for reporting across bounded contexts
- Row-level tenant isolation using `schoolId`

---

## Container Communication

- Web App → Backend API: REST/JSON over HTTP
- Backend API → Database: SQL queries (ORM or query builder)
- Backend internally uses modular communication between contexts
- Tenant onboarding flows are handled by Backend API via TenantManagementModule

---

## Key Design Principle

- Domain logic lives in Backend API
- Web App is purely presentation + interaction layer
- Database is persistence + query optimization layer

## EventsModule

### Core Components

- EventService (orchestrator)
- EventCommandService (write operations)
- EventQueryService (read operations)
- EventTypeStrategy (interface)

### Strategy Implementations

- WebinarStrategy
- ConcertStrategy
- CompetitionStrategy

---

## AuthModule (Component Diagram)

### Components

- AuthService
- TokenService
- RBACService
- TenantResolver

---

## TeachersModule / StudentsModule

### Components

- TeacherQueryService
- TeacherCommandService
- StudentQueryService
- StudentCommandService

---

## LessonPlansModule (Component Diagram)

### Components

- LessonPlanService
- LessonTemplateService

---

## ReportsModule (Component Diagram)

### Components

- ReportService (orchestrator)
- ReportGeneratorStrategy (interface)
- EventReportGenerator
- AcademicReportGenerator
- MixedReportGenerator

## TenantManagementModule (Component Diagram)

### Components

- SchoolService
- TenantIsolationService
