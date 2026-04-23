<!-- EVENT LEVEL ACCESSS -->

# RBAC

Canonical authorization source: `docs/authorization-matrix.md`.
If any conflict appears between this file and the matrix, the matrix wins.

## Roles

- `SYSADMIN`
- `SCHOOL_ADMIN`
- `TEACHER`
- `STUDENT`
- `JURY`

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

В залежності від своєї ролі користувач може або не може виконувати певні дії.
Під час запиту до бекенду у мідлварі перевірятиметься чи має поточний користувач ролі,
які вимагаються для поточного ендпоїнту

Мідлвара `roles` і додається до кожного роуту. Для кожного ендпоїнту передається свій список дозволених ролей.

### ABAC

Проводитиметься централізовано, але динамічно: один middleware-движок + набір перевірок, які підключаються на конкретний роут і передаються у мідлвару, тобто ще до початку бізнес логіки користувачу буде відмовлено або пропущено вперед.

Механізм:

- `authorize({ roles, checks })`
- `roles` - RBAC список ролей дозволених для роута
- `checks` - масив ABAC перевірок (функцій), які виконуються послідовно
- кожна ABAC функція має структуру типу `check(ctx): boolean | Promise<boolean>`
- `ctx` містить дані запиту req і айді юзера для проводження перевірок

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
- `isEventOrganizer(ctx)` - користувач організатор івенту
- `isEventJury(ctx)` - користувач у складі журі івенту
- `canViewGlobalOrTenantEvent(ctx)` - доступ до global або tenant event

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
  - `PATCH /events/:id`: `isEventOrganizer`
  - `POST /events/add-jury-member`: `isEventOrganizer`
  - `POST /events/register`: `isAssignedTeacher` для student + `canViewGlobalOrTenantEvent`
  - `GET /events/:id/participants`: `isEventOrganizer` або `isEventJury`
  - `POST /events/:id/attendance`: `isEventOrganizer`
  - `POST /events/:id/assign-place`: `isEventJury`
  - `POST /events/:id/grade-performance`: `isEventJury`
- Lesson Plans:
  - `POST /lesson-plans/:id/assignments`: `isAssignedTeacher`
- Reports:
  - `POST /reports`: `isOwner`
  - `GET /reports`: `isOwner` або `isSameTenant` (для `SCHOOL_ADMIN`)
  - `GET /reports/:id`: `isOwner` (для `TEACHER`) або `isSameTenant` (для `SCHOOL_ADMIN`)
  - `PATCH /reports/:id`: `isOwner` (для `TEACHER`)
