# RBAC

Canonical authorization source: `docs/authorization-matrix.md`.
If any conflict appears between this file and the matrix, the matrix wins.

## Модель ролей

У системі використовується не один плоский список ролей, а три рівні ролей. Ефективний доступ до endpoint-ів формується з урахуванням рівня платформи, ролі в межах тенанта та, за потреби, ролі в конкретному івенті.

### 1. Platform roles

Ці ролі зберігаються в `User.globalRole` і визначають доступ на рівні всієї платформи.

- `SYSADMIN`
- `MEMBER`

`SYSADMIN` має платформені адміністративні права. `MEMBER` є базовою глобальною роллю для звичайного автентифікованого користувача.

### 2. Tenant roles

Ці ролі зберігаються через `Membership` + `MembershipRole` і діють у межах конкретної школи/тенанта.

- `SCHOOL_ADMIN`
- `TEACHER`
- `STUDENT`

Один користувач має не більше одного `Membership`, але може мати кілька tenant-ролей у межах цього membership.

### 3. Event roles

Ці ролі зберігаються в `EventParticipation.roleInEvent` і діють лише в межах конкретного івенту.

- `PARTICIPANT`
- `PERFORMER`
- `SPEAKER`
- `ORGANIZER`
- `JURY`

Event-role не замінює tenant-role. Наприклад, користувач може бути `TEACHER` у тенанті та `ORGANIZER` або `JURY` у конкретному івенті. Саме тому в доступах до івентів часто використовується комбінація RBAC + ABAC перевірок, а не лише перевірка однієї ролі.

## Як читати доступи

- Platform roles відповідають за глобальні адміністративні можливості та доступ до platform-wide endpoint-ів.
- Tenant roles відповідають за доступ до ресурсів конкретної школи.
- Event roles відповідають за дії в межах конкретного івенту, наприклад оцінювання або перегляд списку учасників.
- Якщо endpoint у таблиці нижче вимагає `JURY`, це означає не окрему системну роль, а наявність відповідної event-role для конкретного івенту.
- Остаточне рішення про доступ ухвалюється як комбінація `roles` + ABAC checks + RLS.

## Tenant Management

| Endpoint                    | Access                     |
| --------------------------- | -------------------------- |
| `POST /tenants`             | `SYSADMIN`                 |
| `GET /tenants`              | `SYSADMIN`                 |
| `GET /tenants/:id`          | `SYSADMIN`, `SCHOOL_ADMIN` |
| `PATCH /tenants/:id/status` | `SYSADMIN`                 |

## Authentication

Усі мають доступ:

- `POST /auth/login`
- `POST /auth/verify-magic-link`

Усі автентифіковані користувачі мають доступ:

- `POST /auth/refresh`
- `POST /auth/logout`
- `GET /auth/me`
- `POST /auth/change-password`

## Identity & Users

| Endpoint         | Access         |
| ---------------- | -------------- |
| `GET /users/all` | `SYSADMIN`     |
| `GET /users`     | `SCHOOL_ADMIN` |
| `POST /users`    | `SCHOOL_ADMIN` |

## 👨‍🏫 Teachers

| Endpoint                  | Access                    |
| ------------------------- | ------------------------- |
| `GET /teachers/:userId`   | `TEACHER`, `SCHOOL_ADMIN` |
| `PATCH /teachers/:userId` | `TEACHER`, `SCHOOL_ADMIN` |

## 🎓 Students

| Endpoint                  | Access                    |
| ------------------------- | ------------------------- |
| `POST /students`          | `TEACHER`                 |
| `GET /students/:userId`   | `TEACHER`, `SCHOOL_ADMIN` |
| `PATCH /students/:userId` | `TEACHER`                 |

## 📅 Events

| Endpoint                            | Access                            |
| ----------------------------------- | --------------------------------- |
| `GET /events`                       | `SCHOOL_ADMIN`, `TEACHER`, `JURY` |
| `GET /events/:id`                   | `SCHOOL_ADMIN`, `TEACHER`, `JURY` |
| `POST /events`                      | `TEACHER`                         |
| `PATCH /events/:id`                 | `TEACHER`                         |
| `POST /events/add-jury-member`      | `TEACHER`                         |
| `POST /events/register`             | `TEACHER`                         |
| `GET /events/:id/participants`      | `TEACHER`, `JURY`                 |
| `POST /events/:id/attendance`       | `TEACHER`                         |
| `POST /events/request-video-upload` | `TEACHER`                         |
| `POST /events/complete-upload`      | `TEACHER`                         |
| `POST /events/assign-place`         | `JURY`                            |
| `POST /events/grade-performance`    | `JURY`                            |

---

## 📚 Lesson Plans

| Endpoint                                      | Access    |
| --------------------------------------------- | --------- |
| `GET /lesson-plans`                           | `TEACHER` |
| `POST /lesson-plans`                          | `TEACHER` |
| `GET /lesson-plans/:id`                       | `TEACHER` |
| `PATCH /lesson-plans/:id`                     | `TEACHER` |
| `POST /lesson-plans/:id/create-from-template` | `TEACHER` |
| `POST /lesson-plans/:id/assignments`          | `TEACHER` |

---

## 📊 Reports

| Endpoint             | Access                    |
| -------------------- | ------------------------- |
| `POST /reports`      | `TEACHER`                 |
| `GET /reports`       | `SCHOOL_ADMIN`, `TEACHER` |
| `GET /reports/:id`   | `SCHOOL_ADMIN`, `TEACHER` |
| `PATCH /reports/:id` | `SCHOOL_ADMIN`            |

---

### RBAC

В залежності від своїх platform-, tenant- та event-ролей користувач може або не може виконувати певні дії.
Під час запиту до бекенду у middleware перевіряється, чи має поточний користувач ролі,
які вимагаються для поточного endpoint-а.

Middleware `roles` додається до кожного роуту. Для кожного endpoint-а передається свій список дозволених effective roles.
Для івентів доступ перевіряється у два кроки: спочатку базова роль, потім роль у конкретному івенті через `EventParticipation`.

### Перевірка ролі в івенті (resource-scoped role check)

Для endpoint-ів `events` і `event participation` роль `ORGANIZER`/`JURY` не повинна перевірятися як абстрактний ABAC predicate без даних.
Перед допуском у handler middleware має:

1. Отримати `eventId` з route params.
2. Зчитати рядок участі користувача в івенті (`EventParticipation`) або дані організатора з `Event`.
3. Перевірити `roleInEvent` для поточного `userId`.
4. Пропустити далі лише якщо роль відповідає вимогам endpoint-а.

Якщо запису участі немає, або роль не підходить, middleware повертає `403`.

### ABAC

Проводитиметься централізовано, але динамічно: один middleware-движок + набір перевірок, які підключаються на конкретний роут і передаються у мідлвару, тобто ще до початку бізнес логіки користувачу буде відмовлено або пропущено вперед.

Механізм:

- `authorize({ roles, checks })`
- `roles` - RBAC список ролей дозволених для роута
- `checks` - масив ABAC перевірок (функцій), які виконуються послідовно
- кожна ABAC функція має структуру типу `check(ctx): boolean | Promise<boolean>`
- `ctx` містить дані запиту req і айді юзера для проводження перевірок

ABAC у цій моделі використовується для контекстних обмежень (`isSameTenant`, `isOwner`, `isAssignedTeacher`), але не як єдине джерело істини для визначення event-role.

Приклад:

```ts
router.patch(
  "/students/:userId",
  authorize({
    roles: ["TEACHER"],
    checks: [isSameTenant, anyOf([isSchoolAdmin, isAssignedTeacher])],
  }),
  handler,
);
```

### ABAC перевірки

- `isSameTenant(ctx)` - перевірка tenant межі
- `isOwner(ctx)` - користувач є власником ресурсу
- `isAssignedTeacher(ctx)` - teacher пов'язаний зі student
- `canViewGlobalOrTenantEvent(ctx)` - доступ до global або tenant event

Перевірки `isEventOrganizer` та `isEventJury` виконуються як resource-scoped role check через lookup у `Event`/`EventParticipation` до входу в бізнес-логіку endpoint-а.

RLS у БД залишається другим рівнем захисту але не замінює middleware-перевірки на API-рівні, адже не має сенсу виконувати логіку до запиту у базу даних, якщо вона його відхилить.

### ABAC приклади

- Tenant Management:
  - `GET /tenants/:id`: `isSelf`/scope check для `SCHOOL_ADMIN`, або лише `SYSADMIN` без ABAC
- Identity & Users:
  - `GET /users`: `isSameTenant`
- Teachers:
  - `GET /teachers/:userId`: `isSameTenant`
  - `PATCH /teachers/:userId`: `isSameTenant`
- Students:
  - `GET /students/:userId`: `isAssignedTeacher` або `isSameTenant` (для `SCHOOL_ADMIN`)
  - `PATCH /students/:userId`: `isAssignedTeacher`
- Events:
  - `GET /events`: `canViewGlobalOrTenantEvent`
  - `GET /events/:id`: `canViewGlobalOrTenantEvent`
  - `PATCH /events/:id`: event-role lookup (`ORGANIZER`)
  - `POST /events/add-jury-member`: event-role lookup (`ORGANIZER`)
  - `POST /events/register`: `isAssignedTeacher` для student + `canViewGlobalOrTenantEvent`
  - `GET /events/:id/participants`: event-role lookup (`ORGANIZER` або `JURY`)
  - `POST /events/:id/attendance`: event-role lookup (`ORGANIZER`)
  - `POST /events/:id/assign-place`: event-role lookup (`JURY`)
  - `POST /events/:id/grade-performance`: event-role lookup (`JURY`)
- Lesson Plans:
  - `POST /lesson-plans/:id/assignments`: `isAssignedTeacher`
- Reports:
  - `POST /reports`: `isOwner`
  - `GET /reports`: `isOwner` або `isSameTenant` (для `SCHOOL_ADMIN`)
  - `GET /reports/:id`: `isOwner` (для `TEACHER`) або `isSameTenant` (для `SCHOOL_ADMIN`)
  - `PATCH /reports/:id`: `isOwner` (для `TEACHER`)
