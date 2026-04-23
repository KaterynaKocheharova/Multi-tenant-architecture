# ERD Diagrams

This document contains Mermaid ER diagrams based on the current data model in `docs/data-model.md`.

## Full Overview

```mermaid
erDiagram
    direction TB
    TENANT {
        id uuid PK
        name text
        status enum
    }

    USER {
        id uuid PK
        fullName text
        email citext
        passwordHash text
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
        schoolId uuid FK
        status enum
    }

    MEMBERSHIP_ROLE {
        id uuid PK
        membershipId uuid FK
        role enum
    }

    MAGIC_LINK {
        id uuid PK
        userId uuid FK
        schoolId uuid FK
        tokenHash text
        expiresAt timestamptz
        consumedAt timestamptz
    }

    TEACHER_STUDENT_ASSIGNMENT {
        id uuid PK
        schoolId uuid FK
        teacherUserId uuid FK
        studentUserId uuid FK
    }

    EVENT {
        id uuid PK
        scope enum
        schoolId uuid FK
        type enum
        name text
        topic text
        videoUrl text
        organizerUserId uuid FK
        startDate timestamptz
        endDate timestamptz
        createdBy uuid FK
    }

    EVENT_PARTICIPATION {
        id uuid PK
        eventId uuid FK
        participantUserId uuid FK
        roleInEvent enum
        attended boolean
        attendanceMarkedAt timestamptz
        notes text
    }

    COMPETITION_PARTICIPATION {
        participationId uuid PK, FK
        grade numeric
        place integer
        createdAt timestamptz
        updatedAt timestamptz
    }

    LESSON_PLAN {
        id uuid PK
        schoolId uuid FK
        teacherUserId uuid FK
        isTemplate boolean
        title text
        content jsonb
    }

    LESSON_PLAN_ASSIGNMENT {
        id uuid PK
        lessonPlanId uuid FK
        schoolId uuid FK
        studentUserId uuid FK
        assignedDate date
        status enum
    }

    REPORT {
        id uuid PK
        schoolId uuid FK
        requestedByUserId uuid FK
        teacherUserId uuid FK
        periodStart date
        periodEnd date
        reportType enum
        filters jsonb
        snapshotPayload jsonb
        generatedAt timestamptz
    }

    USER ||--o| TEACHER_DETAILS : has
    USER ||--o| STUDENT_DETAILS : has
    USER ||--o| MEMBERSHIP : has
    USER ||--o{ MAGIC_LINK : has
    TENANT ||--o{ MEMBERSHIP : has
    TENANT ||--o{ MAGIC_LINK : has
    MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : has

    TENANT ||--o{ TEACHER_STUDENT_ASSIGNMENT : has
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : references_user

    TENANT ||--o{ EVENT : has
    USER ||--o{ EVENT : references_user

    EVENT ||--o{ EVENT_PARTICIPATION : has
    USER ||--o{ EVENT_PARTICIPATION : has
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : has

    TENANT ||--o{ LESSON_PLAN : has
    USER ||--o{ LESSON_PLAN : has
    LESSON_PLAN ||--o{ LESSON_PLAN_ASSIGNMENT : has
    TENANT ||--o{ LESSON_PLAN_ASSIGNMENT : has
    USER ||--o{ LESSON_PLAN_ASSIGNMENT : has

    TENANT ||--o{ REPORT : has
    USER ||--o{ REPORT : references_user
```

## Identity And Tenancy

```mermaid
erDiagram
    direction TB
    TENANT {
        id uuid PK
        name text
        status enum
    }

    USER {
        id uuid PK
        fullName text
        email citext
        passwordHash text
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
        schoolId uuid FK
        status enum
    }

    MEMBERSHIP_ROLE {
        id uuid PK
        membershipId uuid FK
        role enum
    }

    MAGIC_LINK {
        id uuid PK
        userId uuid FK
        schoolId uuid FK
        tokenHash text
        expiresAt timestamptz
        consumedAt timestamptz
    }

    USER ||--o| TEACHER_DETAILS : has
    USER ||--o| STUDENT_DETAILS : has
    USER ||--o| MEMBERSHIP : has
    USER ||--o{ MAGIC_LINK : has
    TENANT ||--o{ MEMBERSHIP : has
    TENANT ||--o{ MAGIC_LINK : has
    MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : has
```

## Events And Participation

```mermaid
erDiagram
    direction TB
    TENANT {
        id uuid PK
        name text
        status enum
    }

    USER {
        id uuid PK
        fullName text
    }

    EVENT {
        id uuid PK
        scope enum
        schoolId uuid FK
        type enum
        name text
        topic text
        videoUrl text
        organizerUserId uuid FK
        startDate timestamptz
        endDate timestamptz
        createdBy uuid FK
    }

    EVENT_PARTICIPATION {
        id uuid PK
        eventId uuid FK
        participantUserId uuid FK
        roleInEvent enum
        attended boolean
        attendanceMarkedAt timestamptz
        notes text
    }

    COMPETITION_PARTICIPATION {
        participationId uuid PK, FK
        grade numeric
        place integer
        createdAt timestamptz
        updatedAt timestamptz
    }

    TENANT ||--o{ EVENT : has
    USER ||--o{ EVENT : references_user
    EVENT ||--o{ EVENT_PARTICIPATION : has
    USER ||--o{ EVENT_PARTICIPATION : has
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : has
```

## Lesson Planning And Reports

```mermaid
erDiagram
    direction TB
    TENANT {
        id uuid PK
        name text
        status enum
    }

    USER {
        id uuid PK
        fullName text
    }

    LESSON_PLAN {
        id uuid PK
        schoolId uuid FK
        teacherUserId uuid FK
        isTemplate boolean
        title text
        content jsonb
    }

    LESSON_PLAN_ASSIGNMENT {
        id uuid PK
        lessonPlanId uuid FK
        schoolId uuid FK
        studentUserId uuid FK
        assignedDate date
        status enum
    }

    REPORT {
        id uuid PK
        schoolId uuid FK
        requestedByUserId uuid FK
        teacherUserId uuid FK
        periodStart date
        periodEnd date
        reportType enum
        filters jsonb
        snapshotPayload jsonb
        generatedAt timestamptz
    }

    TEACHER_STUDENT_ASSIGNMENT {
        id uuid PK
        schoolId uuid FK
        teacherUserId uuid FK
        studentUserId uuid FK
    }

    TENANT ||--o{ LESSON_PLAN : has
    USER ||--o{ LESSON_PLAN : has
    LESSON_PLAN ||--o{ LESSON_PLAN_ASSIGNMENT : has
    TENANT ||--o{ LESSON_PLAN_ASSIGNMENT : has
    USER ||--o{ LESSON_PLAN_ASSIGNMENT : has
    TENANT ||--o{ TEACHER_STUDENT_ASSIGNMENT : has
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : references_user
    TENANT ||--o{ REPORT : has
    USER ||--o{ REPORT : references_user
```
