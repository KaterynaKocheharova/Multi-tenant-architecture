<!-- DOCS_NAV_START -->
[Docs Home](README.md) | [API Design](api-design.md) | [Auth](auth.md) | [RBAC](rbac.md) | [Data Model](data-model.md) | [Security](security.md) | [Deployment](deployment.md) | [Containers](containers.md) | [Context](context.md) | [Frontend](front.md) | [NFR](nfr.md) | [Req-Res Propagation](req-res-propagation.md) | [Risks](risks.md)
<!-- DOCS_NAV_END -->

# RBAC

## Модель ролей

У системі використовується не один список ролей, а три рівні ролей. Ефективний доступ до endpoint-ів формується з урахуванням рівня платформи, ролі в межах тенанта та, за потреби, ролі в конкретному івенті.

### 1. Platform roles

Ці ролі зберігаються в `User.globalRole` і визначають доступ на рівні всієї платформи.

- `SYSADMIN`
- `MEMBER`

`SYSADMIN` має адміністративні права. `MEMBER` є роллю для звичайного автентифікованого користувача.

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

## Як читати доступи

- Platform roles відповідають за глобальні адміністративні можливості.
- Tenant roles відповідають за доступ до ресурсів конкретної школи.
- Event roles відповідають за дії в межах конкретного івенту, наприклад оцінювання.
- Якщо endpoint у таблиці нижче вимагає `JURY`, це означає не окрему системну роль, а наявність відповідної event-role для конкретного івенту.

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

## Teachers

| Endpoint                  | Access                    |
| ------------------------- | ------------------------- |
| `GET /teachers/:userId`   | `TEACHER`, `SCHOOL_ADMIN` |
| `PATCH /teachers/:userId` | `TEACHER`                 |

## Students

| Endpoint                  | Access                    |
| ------------------------- | ------------------------- |
| `POST /students`          | `TEACHER`                 |
| `GET /students/:userId`   | `TEACHER`, `SCHOOL_ADMIN` |
| `PATCH /students/:userId` | `TEACHER`                 |

## Events

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

## Lesson Plans

| Endpoint                                      | Access                    |
| --------------------------------------------- | ------------------------- |
| `GET /lesson-plans`                           | `TEACHER`, `SCHOOL_ADMIN` |
| `POST /lesson-plans`                          | `TEACHER`                 |
| `GET /lesson-plans/:id`                       | `TEACHER`, `SCHOOL_ADMIN` |
| `PATCH /lesson-plans/:id`                     | `TEACHER`                 |
| `POST /lesson-plans/:id/create-from-template` | `TEACHER`                 |
| `POST /lesson-plans/:id/assignments`          | `TEACHER`                 |

## Reports

| Endpoint             | Access                    |
| -------------------- | ------------------------- |
| `POST /reports`      | `TEACHER`                 |
| `GET /reports`       | `SCHOOL_ADMIN`, `TEACHER` |
| `GET /reports/:id`   | `SCHOOL_ADMIN`, `TEACHER` |
| `PATCH /reports/:id` | `SCHOOL_ADMIN`            |

### RBAC

В залежності від своїх platform-, tenant- та event-ролей користувач може або не може виконувати певні дії.
Під час запиту до бекенду у middleware перевіряється, чи має поточний користувач ролі,
які вимагаються для поточного endpoint.

Middleware `roles` додається до кожного роуту. Для кожного endpoint-а передається свій список дозволених effective roles.
Для івентів доступ перевіряється у два кроки: спочатку базова роль, потім роль у конкретному івенті через `EventParticipation`.

### Перевірка ролі в івенті (resource-scoped role check)

Для endpoint-ів `events` і `event participation` роль `ORGANIZER`/`JURY` повинна перевірятися спеціальним способом.
У middleware має:

1. Отримати `eventId` з route params.
2. Зчитати користувача в івенті (`EventParticipation`) або дані організатора з `Event`.
3. Перевірити `roleInEvent` для поточного `userId`.
4. Пропустити далі лише якщо роль відповідає вимогам endpoint-а.

Якщо запису участі немає, або роль не підходить, middleware повертає `403`.

### ABAC

Іноді недостатньо ролі щоб зрозуміти чи користувач може мати доступ (треба перевірити чи належить ресурс юзеру чи ні, учень учителю чи ні і так далі)

## Приклади:

- `authorize({ roles, checks })`
- `roles` - RBAC ролі для роута, передаються у формі {globalRole?: [`MEMBER`, `SYSADMIN`], tenantRole?: [`TEACHER`]}
- `checks` - масив ABAC перевірок
- кожна ABAC функція має структуру типу `check(ctx): boolean | Promise<boolean>`
- `ctx` містить дані запиту req і айді юзера для проводження перевірок

ABAC у цій моделі використовується для обмежень ролі по івенту та тих обмежень що не стосуються ролі (`isOwner`, `isAssignedTeacher`, `isSameTenant`).

Приклад:

```ts
router.patch(
  "/students/:userId",
  authorize({
    roles: [{ tenant: ["TEACHER"] }],
    checks: [isAssignedTeacher],
  }),
  handler,
);
```

### ABAC перевірки

- `isOwner(ctx)` - користувач є власником ресурсу, наприклад - чи учитель є власником плану уроку
- `isAssignedTeacher(ctx)` - teacher пов'язаний зі student
- `isCurrentEventOrganizer`
- `isCurrentEventJury`
- `isCurrentEventParticipant`
