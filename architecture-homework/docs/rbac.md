# 🧩 RBAC

## 📌 Roles

- `SYSADMIN`
- `SCHOOL_ADMIN`
- `TEACHER`
- `STUDENT`
- `JURY`

## 🔐 Tenant Management

| Endpoint                    | Access                                                 |
| --------------------------- | ------------------------------------------------------ |
| `POST /tenants`             | `SYSADMIN`                                             |
| `GET /tenants`              | `SYSADMIN`                                             |
| `GET /tenants/:id`          | `SYSADMIN`, `SCHOOL_ADMIN` (`tenantId` має співпадати) |
| `PATCH /tenants/:id/status` | `SYSADMIN`                                             |

## 🔑 Authentication

Усі мають доступ:

- `POST /auth/login`
- `POST /auth/verify-magic-link`

Усі автентифіковані користувачі мають доступ:

- `POST /auth/refresh`
- `POST /auth/logout`
- `GET /auth/me`
- `POST /auth/change-password`

## 👤 Identity & Users

| Endpoint         | Access                                     |
| ---------------- | ------------------------------------------ |
| `GET /users/all` | `SYSADMIN`                                 |
| `GET /users`     | `SCHOOL_ADMIN` (`tenantId` має співпадати) |
| `POST /users`    | `SCHOOL_ADMIN`                             |

## 👨‍🏫 Teachers

| Endpoint                  | Access                          |
| ------------------------- | ------------------------------- |
| `GET /teachers/:userId`   | `SCHOOL_ADMIN`, або сам учитель |
| `PATCH /teachers/:userId` | `SCHOOL_ADMIN`, або сам учитель |

## 🎓 Students

| Endpoint                  | Access                                   |
| ------------------------- | ---------------------------------------- |
| `POST /students`          | `TEACHER`                                |
| `GET /students/:userId`   | `TEACHER` цього студенту, `SCHOOL_ADMIN` |
| `PATCH /students/:userId` | `TEACHER` цього студенту                 |

## 📅 Events

| Endpoint                             | Access                              |
| ------------------------------------ | ----------------------------------- |
| `GET /events`                        | `SCHOOL_ADMIN` (tenant + global)    |
| `GET /events/:id`                    | ALL (якщо тенант співпадає)         |
| `POST /events`                       | `SCHOOL_ADMIN`, `TEACHER`           |
| `PATCH /events/:id`                  | тільки організатор                  |
| `POST /events/add-jury-member`       | організатор                         |
| `POST /events/register`              | `TEACHER`                           |
| `GET /events/:id/participants`       | організатор, `SCHOOL_ADMIN`, `JURY` |
| `POST /events/:id/attendance`        | організатор                         |
| `POST /events/request-video-upload`  | організатор, `TEACHER` учасник      |
| `POST /events/complete-video-upload` | організатор, `TEACHER` УЧАСНИК      |
| `POST /events/assign-place`          | `JURY`                              |
| `POST /events/grade-performance`     | `JURY`                              |

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

## 🧠 Access Control Model

Система використовує:

### RBAC (Role-Based Access Control)

В залежності від своєї ролі користувач може або не може виконувати певні дії.
Під час запиту до бекенду у мідлварі перевірятиметься чи має поточний користувач ролі,
які вимагаються для поточного ендпоїнту

Мідлвара прийматиме параметр - ролі і додаватиметься до кожного рауту, ці ролі передаватимуться певним списком в залежносіт від ендпоїнту до цієї мідлвари.

### ABAC (Attribute-Based Access Control)

Додаткові перевірки на рівні бізнес-логіки та запитів до бази даних:

- `tenantId` користувача щоб той міг бачити лише ті ресурси які належать школі (наприклад, шкільний адміністратор бачитиме вчителів, учнів, та репорти лише своєї школи, користувачі бачитимуть івенти лише своєї школи, якщо вони не глобальні)
- зв’язки (teacher → students).
  Наприклад, учитель може бачити профайли ат редагувати, додавати плани уроків, записувати на івенти лише своїх студентів.
  Журі може виставляти оцінки учасникам лише івенту, журі якого вони є.
