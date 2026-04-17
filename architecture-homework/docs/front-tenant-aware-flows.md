## Tenant Resolution + Tenant-Scoped API Calls

1. Отримали юзер на фронті.
2. При кожному реквесті беремо тенант айді із стейту де зберігається юзер і передаємо його як динамічний параметр, де він треба

## Post-Login Redirect by Role

| Role           | Redirect after login |
| -------------- | -------------------- |
| `SYSADMIN`     | `/admin/tenants`     |
| `SCHOOL_ADMIN` | `/school/dashboard`  |
| `TEACHER`      | `/home`              |

## Route Protection

Додасться компонент Protected Route. Прийматиме юзер ролі для рауту як пропси і сам компонент

1. Юзер автентифікований? (чи є `access_token`)  
   → Якщо ні, то редірект `/login`
2. Чи юзер має ролі що допускаються для поточного рауту?  
   → Якщо ні, то redirect на `/home`

## Navigation Menu

### SYSADMIN

- Tenants (→ `/admin/tenants`)
- All Users (→ `/admin/users`)
- My Profile (→ `/profile/me`)

### SCHOOL_ADMIN

- Dashboard (→ `/school/dashboard`)
- Users (→ `/school/users`)
- Events (→ `/events`)
- Reports (→ `/reports`)
- My Tenant (→ `/admin/tenants/:id`)
- My Profile (→ `/profile/me`)

### TEACHER

- Events (→ `/events`)
- Students (→ `/students`)
- Lesson Plans (→ `/lesson-plans`)
- Reports (→ `/reports`)
- My Profile (→ `/profile/me` / `/teachers/:userId`)

### JURY

- Events (→ `/events`)
- My Profile (→ `/profile/me`)

---

## Tenant-Aware Scoping per Feature

### Users

- `SCHOOL_ADMIN` calls `GET /users` → sees only users of own tenant.
- `SYSADMIN` calls `GET /users/all` → sees all users across all tenants.

### Events

- `SCHOOL_ADMIN` sees tenant events **and** global events.
- `TEACHER` and `JURY` see only tenant-scoped events.
- Event creation is only available to `TEACHER` within their tenant.

### Students

- `TEACHER` sees only their own assigned students (`TeacherStudentAssignment`).
- `SCHOOL_ADMIN` sees all students in the tenant.

### Lesson Plans

- Fully scoped to `TEACHER`. No cross-teacher visibility.

### Reports

- `TEACHER` sees reports they authored.
- `SCHOOL_ADMIN` sees all reports for the tenant and can update status.

---

## Suspended / Inactive Tenant Handling

- If a user's tenant has `status = suspended`, the backend rejects requests with `403`.
- The frontend should detect this and show a "Your school account has been suspended. Please contact support." screen, blocking all navigation except logout.

---

## Access Token Lifecycle (Client-Side)

| Event                           | Frontend Action                                                                    |
| ------------------------------- | ---------------------------------------------------------------------------------- |
| Successful magic-link verify    | Store `access_token` in memory, redirect by role                                   |
| API returns `401`               | Call `POST /auth/refresh`, retry original request                                  |
| Refresh fails (session expired) | Clear in-memory token, redirect to `/login`                                        |
| User clicks logout              | Call `POST /auth/logout`, clear token, go to `/login`                              |
| Tab / window closed             | `access_token` is lost (in-memory); user re-auths via refresh cookie on next visit |
