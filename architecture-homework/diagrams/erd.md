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
        status enum
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
        createdBy uuid FK
    }

    EVENT_PARTICIPATION {
        id uuid PK
        eventId uuid FK
        participantUserId uuid FK
        schoolId uuid FK
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
        schoolId uuid FK
        participantId uuid FK
        title text
        s3Url text
        issuedAt timestamptz
    }

    LESSON_PLAN {
        id uuid PK
        schoolId uuid FK
        teacherUserId uuid FK
        isTemplate boolean
        title text
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
    TENANT ||--o{ EVENT_PARTICIPATION : has
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : has
    EVENT ||--o{ AWARD : has
    TENANT ||--o{ AWARD : has
    EVENT_PARTICIPATION ||--o{ AWARD : has

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
    TENANT {
        id uuid PK
        name text
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
        schoolId uuid FK
        type enum
        name text
        videoUrl text
        organizerUserId uuid FK
    }

    EVENT_PARTICIPATION {
        id uuid PK
        eventId uuid FK
        participantUserId uuid FK
        schoolId uuid FK
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
        schoolId uuid FK
        participantId uuid FK
        title text
        s3Url text
    }

    TENANT ||--o{ EVENT : has
    USER ||--o{ EVENT : references_user
    EVENT ||--o{ EVENT_PARTICIPATION : has
    USER ||--o{ EVENT_PARTICIPATION : has
    TENANT ||--o{ EVENT_PARTICIPATION : has
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : has
    EVENT ||--o{ AWARD : has
    EVENT_PARTICIPATION ||--o{ AWARD : has
    TENANT ||--o{ AWARD : has
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
        schoolId uuid FK
        teacherUserId uuid FK
        isTemplate boolean
        title text
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
