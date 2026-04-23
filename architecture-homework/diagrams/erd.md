# ERD-діаграми

Цей документ містить Mermaid ER-діаграми, побудовані на основі актуальної моделі даних з `docs/data-model.md`.

## Повний огляд

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

    USER ||--o| TEACHER_DETAILS : має
    USER ||--o| STUDENT_DETAILS : має
    USER ||--o| MEMBERSHIP : має
    USER ||--o{ MAGIC_LINK : має
    TENANT ||--o{ MEMBERSHIP : має
    TENANT ||--o{ MAGIC_LINK : має
    MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : має

    TENANT ||--o{ TEACHER_STUDENT_ASSIGNMENT : має
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : посилання_на_користувача

    TENANT ||--o{ EVENT : має
    USER ||--o{ EVENT : посилання_на_користувача

    EVENT ||--o{ EVENT_PARTICIPATION : має
    USER ||--o{ EVENT_PARTICIPATION : має
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : має

    TENANT ||--o{ LESSON_PLAN : має
    USER ||--o{ LESSON_PLAN : має
    LESSON_PLAN ||--o{ LESSON_PLAN_ASSIGNMENT : має
    TENANT ||--o{ LESSON_PLAN_ASSIGNMENT : має
    USER ||--o{ LESSON_PLAN_ASSIGNMENT : має

    TENANT ||--o{ REPORT : має
    USER ||--o{ REPORT : посилання_на_користувача
```

## Ідентифікація та тенанти

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

    USER ||--o| TEACHER_DETAILS : має
    USER ||--o| STUDENT_DETAILS : має
    USER ||--o| MEMBERSHIP : має
    USER ||--o{ MAGIC_LINK : має
    TENANT ||--o{ MEMBERSHIP : має
    TENANT ||--o{ MAGIC_LINK : має
    MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : має
```

## Події та участь

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

    TENANT ||--o{ EVENT : має
    USER ||--o{ EVENT : посилання_на_користувача
    EVENT ||--o{ EVENT_PARTICIPATION : має
    USER ||--o{ EVENT_PARTICIPATION : має
    EVENT_PARTICIPATION ||--|| COMPETITION_PARTICIPATION : має
```

## Планування уроків та звітність

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

    TENANT ||--o{ LESSON_PLAN : має
    USER ||--o{ LESSON_PLAN : має
    LESSON_PLAN ||--o{ LESSON_PLAN_ASSIGNMENT : має
    TENANT ||--o{ LESSON_PLAN_ASSIGNMENT : має
    USER ||--o{ LESSON_PLAN_ASSIGNMENT : має
    TENANT ||--o{ TEACHER_STUDENT_ASSIGNMENT : має
    USER ||--o{ TEACHER_STUDENT_ASSIGNMENT : посилання_на_користувача
    TENANT ||--o{ REPORT : має
    USER ||--o{ REPORT : посилання_на_користувача
```
