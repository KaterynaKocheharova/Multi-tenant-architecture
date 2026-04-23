# Модель даних

## Перелік сутностей

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

## Таблиці

## Контекст управління тенантами

### Tenant

| Стовпець  | Тип                               | Обмеження |
| --------- | --------------------------------- | --------- |
| id        | uuid                              | PK        |
| name      | text                              | not null  |
| status    | enum(active, suspended, archived) | not null  |
| createdAt | timestamptz                       | not null  |
| updatedAt | timestamptz                       | not null  |

## Контекст ідентифікації

### User

| Стовпець     | Тип                    | Обмеження               |
| ------------ | ---------------------- | ----------------------- |
| id           | uuid                   | PK                      |
| fullName     | text                   | not null                |
| email        | citext                 | unique, not null        |
| passwordHash | text                   | not null                |
| globalRole   | enum(Sysadmin, Member) | not null default Member |
| isActive     | boolean                | not null default true   |
| createdAt    | timestamptz            | not null                |
| updatedAt    | timestamptz            | not null                |

Примітки:

- `globalRole` визначає доступ на рівні платформи - їх дві - `SYSADMIN` та `MEMBER`.
- Сирі паролі не зберігаються; персиститься лише `passwordHash`.

### TeacherDetails

| Стовпець          | Тип         | Обмеження                       |
| ----------------- | ----------- | ------------------------------- |
| id                | uuid        | PK                              |
| userId            | uuid        | FK -> User.id, unique, not null |
| musicalInstrument | text        | null                            |
| createdAt         | timestamptz | not null                        |
| updatedAt         | timestamptz | not null                        |

### StudentDetails

| Стовпець          | Тип         | Обмеження                       |
| ----------------- | ----------- | ------------------------------- |
| id                | uuid        | PK                              |
| userId            | uuid        | FK -> User.id, unique, not null |
| musicalInstrument | text        | null                            |
| classNumber       | text        | null                            |
| createdAt         | timestamptz | not null                        |
| updatedAt         | timestamptz | not null                        |

Примітки:

- Деталі профілю за ролями винесені в `TeacherDetails` і `StudentDetails`. Зберігатимуть публічну інформацію.

### TeacherStudentAssignment

Призначення: зв'язок викладача з учнем у межах одного тенанта.

| Стовпець      | Тип         | Обмеження                 |
| ------------- | ----------- | ------------------------- |
| id            | uuid        | PK                        |
| schoolId      | uuid        | FK -> Tenant.id, not null |
| teacherUserId | uuid        | FK -> User.id, not null   |
| studentUserId | uuid        | FK -> User.id, not null   |
| createdAt     | timestamptz | not null                  |
| updatedAt     | timestamptz | not null                  |

Обмеження:

- unique(schoolId, teacherUserId, studentUserId)
- teacherUserId != studentUserId

### Membership

Призначення: пов'язує користувача рівно з одним тенантом.

| Стовпець  | Тип                     | Обмеження                       |
| --------- | ----------------------- | ------------------------------- |
| id        | uuid                    | PK                              |
| userId    | uuid                    | FK -> User.id, unique, not null |
| schoolId  | uuid                    | FK -> Tenant.id, not null       |
| status    | enum(active, suspended) | not null default active         |
| createdAt | timestamptz             | not null                        |
| updatedAt | timestamptz             | not null                        |

Примітки:

- Облікові записи Sysadmin можуть існувати без membership.
- Усі користувачі школи повинні мати один рядок membership.
- Основні ролі: `SCHOOL_ADMIN`, `TEACHER`, `STUDENT`, `JURY`.
- Один member - одне membership.
- Міжшкільна співпраця моделюється через участь у глобальних івентах, а не через membership у кількох тенантах.

### MembershipRole

| Стовпець     | Тип                                 | Обмеження                     |
| ------------ | ----------------------------------- | ----------------------------- |
| id           | uuid                                | PK                            |
| membershipId | uuid                                | FK -> Membership.id, not null |
| role         | enum(schoolAdmin, teacher, student) | not null                      |
| createdAt    | timestamptz                         | not null                      |

Обмеження:

- unique(membershipId, role)
- проте один корстувач може мати кілька ролей

### MagicLink

Призначення: зберігає одноразові посилання для автентифікації.

| Стовпець   | Тип         | Обмеження                 |
| ---------- | ----------- | ------------------------- |
| id         | uuid        | PK                        |
| userId     | uuid        | FK -> User.id, not null   |
| schoolId   | uuid        | FK -> Tenant.id, not null |
| tokenHash  | text        | unique, not null          |
| expiresAt  | timestamptz | not null                  |
| consumedAt | timestamptz | null                      |
| createdAt  | timestamptz | not null                  |

Примітки:

- сирі токени magic-link ніколи не зберігаються
- токен валідний лише один раз
- прострочені або використані посилання мають бути відхилені під час верифікації

## Контекст управління подіями

### Event

| Стовпець        | Тип                                 | Обмеження               |
| --------------- | ----------------------------------- | ----------------------- |
| id              | uuid                                | PK                      |
| scope           | enum(TENANT, GLOBAL)                | not null                |
| schoolId        | uuid                                | FK -> Tenant.id, null   |
| type            | enum(webinar, concert, competition) | not null                |
| name            | text                                | not null                |
| topic           | text                                | null                    |
| videoUrl        | text                                | null                    |
| organizerUserId | uuid                                | FK -> User.id, not null |
| startDate       | timestamptz                         | not null                |
| endDate         | timestamptz                         | not null                |
| createdBy       | uuid                                | FK -> User.id, not null |
| createdAt       | timestamptz                         | not null                |
| updatedAt       | timestamptz                         | not null                |

Обмеження:

- check(endDate >= startDate)
- check((scope = 'TENANT' and schoolId is not null) or (scope = 'GLOBAL' and schoolId is null))
- `schoolId`: тенант-власник видимості для подій у межах тенанта, null для глобальних подій.

### EventParticipation

| Стовпець           | Тип                                                    | Обмеження                    |
| ------------------ | ------------------------------------------------------ | ---------------------------- |
| id                 | uuid                                                   | PK                           |
| eventId            | uuid                                                   | FK -> Event.id, not null     |
| participantUserId  | uuid                                                   | FK -> User.id, not null      |
| roleInEvent        | enum(participant, performer, speaker, organizer, jury) | not null default participant |
| attended           | boolean                                                | not null default false       |
| attendanceMarkedAt | timestamptz                                            | null                         |
| notes              | text                                                   | null                         |
| createdAt          | timestamptz                                            | not null                     |
| updatedAt          | timestamptz                                            | not null                     |

Обмеження:

- unique(eventId, participantUserId)

- Додаткові ролі в межах івенту: `PARTICIPANT`, `PERFORMER`, `SPEAKER`, `ORGANIZER`, `JURY`

### CompetitionParticipation

| Стовпець        | Тип          | Обмеження                   |
| --------------- | ------------ | --------------------------- |
| participationId | uuid         | FK -> EventParticipation.id |
| grade           | numeric(5,2) | null                        |
| place           | integer      | null                        |
| createdAt       | timestamptz  | not null                    |
| updatedAt       | timestamptz  | not null                    |

Правило валідації:

- participation має посилатися на Event, де type = competition.

## Контекст планування уроків

### LessonPlan

| Стовпець      | Тип         | Обмеження                 |
| ------------- | ----------- | ------------------------- |
| id            | uuid        | PK                        |
| schoolId      | uuid        | FK -> Tenant.id, not null |
| teacherUserId | uuid        | FK -> User.id, not null   |
| isTemplate    | boolean     | not null default false    |
| title         | text        | null                      |
| content       | jsonb       | not null                  |
| createdAt     | timestamptz | not null                  |
| updatedAt     | timestamptz | not null                  |

### LessonPlanAssignment

Призначення: пов'язує план із конкретним учнем і розкладом.

| Стовпець      | Тип                                 | Обмеження                     |
| ------------- | ----------------------------------- | ----------------------------- |
| id            | uuid                                | PK                            |
| lessonPlanId  | uuid                                | FK -> LessonPlan.id, not null |
| schoolId      | uuid                                | FK -> Tenant.id, not null     |
| studentUserId | uuid                                | FK -> User.id, not null       |
| assignedDate  | date                                | not null                      |
| status        | enum(planned, completed, cancelled) | not null default planned      |
| notes         | text                                | null                          |
| createdAt     | timestamptz                         | not null                      |
| updatedAt     | timestamptz                         | not null                      |

Обмеження:

- unique(lessonPlanId, studentUserId, assignedDate) - один план урока на урок, не може бути декілька планів на один день на той самий урок

## Контекст звітності

### Report

| Стовпець          | Тип                                  | Обмеження                 |
| ----------------- | ------------------------------------ | ------------------------- |
| id                | uuid                                 | PK                        |
| schoolId          | uuid                                 | FK -> Tenant.id, not null |
| requestedByUserId | uuid                                 | FK -> User.id, not null   |
| teacherUserId     | uuid                                 | FK -> User.id, null       |
| periodStart       | date                                 | not null                  |
| periodEnd         | date                                 | not null                  |
| reportType        | enum(annual, semester, term, random) | not null                  |
| filters           | jsonb                                | null                      |
| snapshotPayload   | jsonb                                | not null                  |
| generatedAt       | timestamptz                          | not null                  |

Обмеження:

- check(periodEnd >= periodStart)
