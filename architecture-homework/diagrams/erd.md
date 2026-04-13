# ERD Diagrams

This document contains Mermaid ER diagrams based on the current data model in `docs/data-model.md`.

## Full Overview

```mermaid
erDiagram
    TENANT {
        id uuid PK
        name text
        status enum
    }

    USER {
        id uuid PK
        fullName text
        email citext
        globalRole enum
        isActive boolean
    }

    TEACHER_DETAILS {
        id uuid PK
        userId uuid FK
        musicalInstrument text
    }

    STUDENT_DETAILS {
        id uuid PK
        userId uuid FK
        musicalInstrument text
        classNumber text
    }

    MEMBERSHIP {
        id uuid PK
        userId uuid FK
        tenantId uuid FK
        status enum
    }

    MEMBERSHIP_ROLE {
        id uuid PK
        membershipId uuid FK
        role enum
    }

    TEACHER_STUDENT_ASSIGNMENT {
        id uuid PK
        tenantId uuid FK
        teacherUserId uuid FK
        studentUserId uuid FK
        status enum
    }

    EVENT {
        id uuid PK
        scope enum
        tenantId uuid FK
        type enum
        name text
        topic text
        videoUrl text
        organizerUserId uuid FK
        createdBy uuid FK
    }

    EVENT_PARTICIPATION {
        id uuid PK
        eventId uuid FK
        participantUserId uuid FK
        tenantId uuid FK
        roleInEvent enum
        attended boolean
    }

    COMPETITION_PARTICIPATION {
        participationId uuid PK, FK
        grade numeric
        place integer
        juryNotes text
    }

    AWARD {
        id uuid PK
        eventId uuid FK
        tenantId uuid FK
        participantId uuid FK
        title text
        s3Url text
        issuedAt timestamptz
    }

    LESSON_PLAN {
        id uuid PK
        tenantId uuid FK
        teacherUserId uuid FK
        isTemplate boolean
        title text
    }

    LESSON_PLAN_ASSIGNMENT {
        id uuid PK
        lessonPlanId uuid FK
        tenantId uuid FK
        studentUserId uuid FK
        assignedDate date
        status enum
    }

    REPORT {
        id uuid PK
        tenantId uuid FK
        requestedByUserId uuid FK
        teacherUserId uuid FK
        periodStart date
        periodEnd date
        reportType enum
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
        id uuid PK
        name text
    }

    USER {
        id uuid PK
        fullName text
        email citext
        globalRole enum
    }

    TEACHER_DETAILS {
        id uuid PK
        userId uuid FK
        musicalInstrument text
    }

    STUDENT_DETAILS {
        id uuid PK
        userId uuid FK
        musicalInstrument text
        classNumber text
    }

    MEMBERSHIP {
        id uuid PK
        userId uuid FK
        tenantId uuid FK
    }

    MEMBERSHIP_ROLE {
        id uuid PK
        membershipId uuid FK
        role enum
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
        id uuid PK
        name text
    }

    USER {
        id uuid PK
        fullName text
    }

    EVENT {
        id uuid PK
        scope enum
        tenantId uuid FK
        type enum
        name text
        videoUrl text
        organizerUserId uuid FK
    }

    EVENT_PARTICIPATION {
        id uuid PK
        eventId uuid FK
        participantUserId uuid FK
        tenantId uuid FK
        roleInEvent enum
        attended boolean
    }

    COMPETITION_PARTICIPATION {
        participationId uuid PK, FK
        grade numeric
        place integer
    }

    AWARD {
        id uuid PK
        eventId uuid FK
        tenantId uuid FK
        participantId uuid FK
        title text
        s3Url text
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
        id uuid PK
        name text
    }

    USER {
        id uuid PK
        fullName text
    }

    LESSON_PLAN {
        id uuid PK
        tenantId uuid FK
        teacherUserId uuid FK
        isTemplate boolean
        title text
    }

    LESSON_PLAN_ASSIGNMENT {
        id uuid PK
        lessonPlanId uuid FK
        tenantId uuid FK
        studentUserId uuid FK
        assignedDate date
        status enum
    }

    REPORT {
        id uuid PK
        tenantId uuid FK
        requestedByUserId uuid FK
        teacherUserId uuid FK
        periodStart date
        periodEnd date
        reportType enum
    }

    TEACHER_STUDENT_ASSIGNMENT {
        id uuid PK
        tenantId uuid FK
        teacherUserId uuid FK
        studentUserId uuid FK
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
