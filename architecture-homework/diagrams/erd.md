# ERD Diagrams

This document contains Mermaid ER diagrams based on the current data model in `docs/data-model.md`.

## Full Overview

```mermaid
erDiagram
    TENANT {
        uuid id PK
        text name
        enum status
    }

    USER {
        uuid id PK
        text fullName
        citext email
        enum globalRole
        boolean isActive
    }

    TEACHER_DETAILS {
        uuid id PK
        uuid userId FK
        text musicalInstrument
    }

    STUDENT_DETAILS {
        uuid id PK
        uuid userId FK
        text musicalInstrument
        text classNumber
    }

    MEMBERSHIP {
        uuid id PK
        uuid userId FK
        uuid tenantId FK
        enum status
    }

    MEMBERSHIP_ROLE {
        uuid id PK
        uuid membershipId FK
        enum role
    }

    TEACHER_STUDENT_ASSIGNMENT {
        uuid id PK
        uuid tenantId FK
        uuid teacherUserId FK
        uuid studentUserId FK
        enum status
    }

    EVENT {
        uuid id PK
        enum scope
        uuid tenantId FK
        enum type
        text name
        text topic
        text videoUrl
        uuid organizerUserId FK
        uuid createdBy FK
    }

    EVENT_PARTICIPATION {
        uuid id PK
        uuid eventId FK
        uuid participantUserId FK
        uuid tenantId FK
        enum roleInEvent
        boolean attended
    }

    COMPETITION_PARTICIPATION {
        uuid participationId PK, FK
        numeric grade
        integer place
        text juryNotes
    }

    AWARD {
        uuid id PK
        uuid eventId FK
        uuid tenantId FK
        uuid participantId FK
        text title
        text s3Url
        timestamptz issuedAt
    }

    LESSON_PLAN {
        uuid id PK
        uuid tenantId FK
        uuid teacherUserId FK
        boolean isTemplate
        text title
    }

    LESSON_PLAN_ASSIGNMENT {
        uuid id PK
        uuid lessonPlanId FK
        uuid tenantId FK
        uuid studentUserId FK
        date assignedDate
        enum status
    }

    REPORT {
        uuid id PK
        uuid tenantId FK
        uuid requestedByUserId FK
        uuid teacherUserId FK
        date periodStart
        date periodEnd
        enum reportType
    }

    USER ||--o| TEACHER_DETAILS : has
    USER ||--o| STUDENT_DETAILS : has
    USER ||--o| MEMBERSHIP : belongs_to
    TENANT ||--o{ MEMBERSHIP : has
    MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : grants

    TENANT ||--o{ TEACHER_STUDENT_ASSIGNMENT : scopes
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : teacher
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : student

    TENANT ||--o{ EVENT : owns_tenant_events
    USER ||--o{ EVENT : organizes
    USER ||--o{ EVENT : creates

    EVENT ||--o{ EVENT_PARTICIPATION : has
    USER ||--o{ EVENT_PARTICIPATION : joins
    TENANT ||--o{ EVENT_PARTICIPATION : scopes
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : extends
    EVENT ||--o{ AWARD : results_in
    TENANT ||--o{ AWARD : scopes
    EVENT_PARTICIPATION ||--o{ AWARD : receives

    TENANT ||--o{ LESSON_PLAN : scopes
    USER ||--o{ LESSON_PLAN : authors
    LESSON_PLAN ||--o{ LESSON_PLAN_ASSIGNMENT : assigned_as
    TENANT ||--o{ LESSON_PLAN_ASSIGNMENT : scopes
    USER ||--o{ LESSON_PLAN_ASSIGNMENT : assigned_to

    TENANT ||--o{ REPORT : owns
    USER ||--o{ REPORT : requested_by
    USER ||--o{ REPORT : filtered_for
```

## Identity And Tenancy

```mermaid
erDiagram
    TENANT {
        uuid id PK
        text name
    }

    USER {
        uuid id PK
        text fullName
        citext email
        enum globalRole
    }

    TEACHER_DETAILS {
        uuid id PK
        uuid userId FK
        text musicalInstrument
    }

    STUDENT_DETAILS {
        uuid id PK
        uuid userId FK
        text musicalInstrument
        text classNumber
    }

    MEMBERSHIP {
        uuid id PK
        uuid userId FK
        uuid tenantId FK
    }

    MEMBERSHIP_ROLE {
        uuid id PK
        uuid membershipId FK
        enum role
    }

    USER ||--o| TEACHER_DETAILS : has
    USER ||--o| STUDENT_DETAILS : has
    USER ||--o| MEMBERSHIP : belongs_to
    TENANT ||--o{ MEMBERSHIP : has
    MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : grants
```

## Events And Participation

```mermaid
erDiagram
    TENANT {
        uuid id PK
        text name
    }

    USER {
        uuid id PK
        text fullName
    }

    EVENT {
        uuid id PK
        enum scope
        uuid tenantId FK
        enum type
        text name
        text videoUrl
        uuid organizerUserId FK
    }

    EVENT_PARTICIPATION {
        uuid id PK
        uuid eventId FK
        uuid participantUserId FK
        uuid tenantId FK
        enum roleInEvent
        boolean attended
    }

    COMPETITION_PARTICIPATION {
        uuid participationId PK, FK
        numeric grade
        integer place
    }

    AWARD {
        uuid id PK
        uuid eventId FK
        uuid tenantId FK
        uuid participantId FK
        text title
        text s3Url
    }

    TENANT ||--o{ EVENT : owns_tenant_events
    USER ||--o{ EVENT : organizes
    EVENT ||--o{ EVENT_PARTICIPATION : has
    USER ||--o{ EVENT_PARTICIPATION : joins
    TENANT ||--o{ EVENT_PARTICIPATION : scopes
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : extends
    EVENT ||--o{ AWARD : results_in
    EVENT_PARTICIPATION ||--o{ AWARD : receives
    TENANT ||--o{ AWARD : scopes
```

## Lesson Planning And Reports

```mermaid
erDiagram
    TENANT {
        uuid id PK
        text name
    }

    USER {
        uuid id PK
        text fullName
    }

    LESSON_PLAN {
        uuid id PK
        uuid tenantId FK
        uuid teacherUserId FK
        boolean isTemplate
        text title
    }

    LESSON_PLAN_ASSIGNMENT {
        uuid id PK
        uuid lessonPlanId FK
        uuid tenantId FK
        uuid studentUserId FK
        date assignedDate
        enum status
    }

    REPORT {
        uuid id PK
        uuid tenantId FK
        uuid requestedByUserId FK
        uuid teacherUserId FK
        date periodStart
        date periodEnd
        enum reportType
    }

    TEACHER_STUDENT_ASSIGNMENT {
        uuid id PK
        uuid tenantId FK
        uuid teacherUserId FK
        uuid studentUserId FK
    }

    TENANT ||--o{ LESSON_PLAN : scopes
    USER ||--o{ LESSON_PLAN : authors
    LESSON_PLAN ||--o{ LESSON_PLAN_ASSIGNMENT : assigned_as
    TENANT ||--o{ LESSON_PLAN_ASSIGNMENT : scopes
    USER ||--o{ LESSON_PLAN_ASSIGNMENT : assigned_to
    TENANT ||--o{ TEACHER_STUDENT_ASSIGNMENT : scopes
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : teacher
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : student
    TENANT ||--o{ REPORT : owns
    USER ||--o{ REPORT : requested_by
    USER ||--o{ REPORT : filtered_for
```