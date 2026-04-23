# Data Model

## Entities

1. Tenant
2. User
3. TeacherDetails
4. StudentDetails
5. Membership
6. MembershipRole
7. TeacherStudentAssignment
8. Event
9. EventParticipation
10. CompetitionParticipation
11. LessonPlan
12. LessonPlanAssignment
13. Report
14. MagicLink

## Tables

### Tenant

| Column    | Type                              | Constraints |
| --------- | --------------------------------- | ----------- |
| id        | uuid                              | PK          |
| name      | text                              | not null    |
| status    | enum(active, suspended, archived) | not null    |
| createdAt | timestamptz                       | not null    |
| updatedAt | timestamptz                       | not null    |

### User

| Column       | Type                   | Constraints             |
| ------------ | ---------------------- | ----------------------- |
| id           | uuid                   | PK                      |
| fullName     | text                   | not null                |
| email        | citext                 | unique, not null        |
| passwordHash | text                   | not null                |
| globalRole   | enum(Sysadmin, Member) | not null default Member |
| isActive     | boolean                | not null default true   |
| createdAt    | timestamptz            | not null                |
| updatedAt    | timestamptz            | not null                |

Notes:

- `User` stores platform identity and password credential hash.
- `globalRole` defines platform-level access outside tenant-scoped roles.
- Raw passwords are never stored; only `passwordHash` is persisted.
- Role-specific profile details are moved to `TeacherDetails` and `StudentDetails`.

### TeacherDetails

| Column            | Type        | Constraints                     |
| ----------------- | ----------- | ------------------------------- |
| id                | uuid        | PK                              |
| userId            | uuid        | FK -> User.id, unique, not null |
| musicalInstrument | text        | null                            |
| createdAt         | timestamptz | not null                        |
| updatedAt         | timestamptz | not null                        |

### StudentDetails

| Column            | Type        | Constraints                     |
| ----------------- | ----------- | ------------------------------- |
| id                | uuid        | PK                              |
| userId            | uuid        | FK -> User.id, unique, not null |
| musicalInstrument | text        | null                            |
| classNumber       | text        | null                            |
| createdAt         | timestamptz | not null                        |
| updatedAt         | timestamptz | not null                        |

### Membership

Purpose: links user to exactly one tenant.

| Column    | Type                    | Constraints                     |
| --------- | ----------------------- | ------------------------------- |
| id        | uuid                    | PK                              |
| userId    | uuid                    | FK -> User.id, unique, not null |
| tenantId  | uuid                    | FK -> Tenant.id, not null       |
| status    | enum(active, suspended) | not null default active         |
| createdAt | timestamptz             | not null                        |
| updatedAt | timestamptz             | not null                        |

Notes:

- Sysadmin accounts can exist without membership.
- All school users must have one membership row.
- Identity policy (v1): school-scoped users (`SCHOOL_ADMIN`, `TEACHER`, `STUDENT`, `JURY`) have one membership and belong to one tenant. Cross-school collaboration is modeled in event participation, not multi-tenant membership.- Identity policy (v1): school-scoped users (`SCHOOL_ADMIN`, `TEACHER`, `STUDENT`, `JURY`) have one membership and belong to one tenant. Cross-school collaboration is modeled in event participation, not multi-tenant membership.

### MembershipRole

Purpose: stores one or more roles assigned to a membership.

| Column       | Type                                | Constraints                   |
| ------------ | ----------------------------------- | ----------------------------- |
| id           | uuid                                | PK                            |
| membershipId | uuid                                | FK -> Membership.id, not null |
| role         | enum(schoolAdmin, teacher, student) | not null                      |
| createdAt    | timestamptz                         | not null                      |

Constraints:

- unique(membershipId, role)

### MagicLink

Purpose: stores one-time passwordless authentication links.

| Column     | Type        | Constraints               |
| ---------- | ----------- | ------------------------- |
| id         | uuid        | PK                        |
| userId     | uuid        | FK -> User.id, not null   |
| tenantId   | uuid        | FK -> Tenant.id, not null |
| tokenHash  | text        | unique, not null          |
| expiresAt  | timestamptz | not null                  |
| consumedAt | timestamptz | null                      |
| createdAt  | timestamptz | not null                  |

Notes:

- raw magic-link tokens are never stored
- token is valid only once
- expired or consumed links must be rejected during verification

### TeacherStudentAssignment

Purpose: teacher to student mapping within one tenant.

| Column        | Type                   | Constraints               |
| ------------- | ---------------------- | ------------------------- |
| id            | uuid                   | PK                        |
| tenantId      | uuid                   | FK -> Tenant.id, not null |
| teacherUserId | uuid                   | FK -> User.id, not null   |
| studentUserId | uuid                   | FK -> User.id, not null   |
| status        | enum(active, inactive) | not null default active   |
| createdAt     | timestamptz            | not null                  |
| updatedAt     | timestamptz            | not null                  |

Constraints:

- unique(tenantId, teacherUserId, studentUserId)
- teacherUserId != studentUserId

### Event

| Column             | Type                                | Constraints               |
| ------------------ | ----------------------------------- | ------------------------- |
| id                 | uuid                                | PK                        |
| scope              | enum(TENANT, GLOBAL)                | not null                  |
| tenantId           | uuid                                | FK -> Tenant.id, null     |
| organizingTenantId | uuid                                | FK -> Tenant.id, not null |
| type               | enum(webinar, concert, competition) | not null                  |
| name               | text                                | not null                  |
| topic              | text                                | null                      |
| videoUrl           | text                                | null                      |
| organizerUserId    | uuid                                | FK -> User.id, not null   |
| startDate          | timestamptz                         | not null                  |
| endDate            | timestamptz                         | not null                  |
| createdBy          | uuid                                | FK -> User.id, not null   |
| createdAt          | timestamptz                         | not null                  |
| updatedAt          | timestamptz                         | not null                  |

Constraints:

- check(endDate >= startDate)
- check((scope = 'TENANT' and tenantId is not null) or (scope = 'GLOBAL' and tenantId is null))

Attribution semantics:

- `tenantId`: visibility owner for tenant-scoped events, null for global events.
- `organizingTenantId`: tenant that created/owns organization narrative for the event.

### EventParticipation

| Column               | Type                                             | Constraints                  |
| -------------------- | ------------------------------------------------ | ---------------------------- |
| id                   | uuid                                             | PK                           |
| eventId              | uuid                                             | FK -> Event.id, not null     |
| participantUserId    | uuid                                             | FK -> User.id, not null      |
| participantTenantId  | uuid                                             | FK -> Tenant.id, not null    |
| registeredByTenantId | uuid                                             | FK -> Tenant.id, not null    |
| roleInEvent          | enum(participant, performer, speaker, organizer) | not null default participant |
| attended             | boolean                                          | not null default false       |
| attendanceMarkedAt   | timestamptz                                      | null                         |
| notes                | text                                             | null                         |
| createdAt            | timestamptz                                      | not null                     |
| updatedAt            | timestamptz                                      | not null                     |

Constraints:

- unique(eventId, participantUserId)

Attribution semantics:

- `participantTenantId`: participant home tenant.
- `registeredByTenantId`: tenant that performed the registration action.

### CompetitionParticipation

| Column          | Type         | Constraints                     |
| --------------- | ------------ | ------------------------------- |
| participationId | uuid         | PK, FK -> EventParticipation.id |
| grade           | numeric(5,2) | null                            |
| place           | integer      | null                            |
| juryNotes       | text         | null                            |
| createdAt       | timestamptz  | not null                        |
| updatedAt       | timestamptz  | not null                        |

Validation rule:

- participation must reference an Event where type = competition.

### LessonPlan

| Column        | Type        | Constraints               |
| ------------- | ----------- | ------------------------- |
| id            | uuid        | PK                        |
| tenantId      | uuid        | FK -> Tenant.id, not null |
| teacherUserId | uuid        | FK -> User.id, not null   |
| isTemplate    | boolean     | not null default false    |
| title         | text        | null                      |
| content       | jsonb       | not null                  |
| createdAt     | timestamptz | not null                  |
| updatedAt     | timestamptz | not null                  |

### LessonPlanAssignment

Purpose: links a plan to a specific student and schedule.

| Column        | Type                                | Constraints                   |
| ------------- | ----------------------------------- | ----------------------------- |
| id            | uuid                                | PK                            |
| lessonPlanId  | uuid                                | FK -> LessonPlan.id, not null |
| tenantId      | uuid                                | FK -> Tenant.id, not null     |
| studentUserId | uuid                                | FK -> User.id, not null       |
| assignedDate  | date                                | not null                      |
| status        | enum(planned, completed, cancelled) | not null default planned      |
| notes         | text                                | null                          |
| createdAt     | timestamptz                         | not null                      |
| updatedAt     | timestamptz                         | not null                      |

Constraints:

- unique(lessonPlanId, studentUserId, assignedDate)

### Report

| Column            | Type                                 | Constraints               |
| ----------------- | ------------------------------------ | ------------------------- |
| id                | uuid                                 | PK                        |
| tenantId          | uuid                                 | FK -> Tenant.id, not null |
| requestedByUserId | uuid                                 | FK -> User.id, not null   |
| teacherUserId     | uuid                                 | FK -> User.id, null       |
| periodStart       | date                                 | not null                  |
| periodEnd         | date                                 | not null                  |
| reportType        | enum(annual, semester, term, random) | not null                  |
| filters           | jsonb                                | null                      |
| snapshotPayload   | jsonb                                | not null                  |
| generatedAt       | timestamptz                          | not null                  |

Constraints:

- check(periodEnd >= periodStart)
