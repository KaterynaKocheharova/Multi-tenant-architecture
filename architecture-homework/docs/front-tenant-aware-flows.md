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

## Tenant-Aware Scoping

## Access Token Lifecycle

1. При переході лінкою у імейлі запит на верифікацію до бека, який дає токен.
2. Токен зберігаємо in-memory, не в локал сторідж щоб не можна було вкрасти.
3. При кожному запиті токен додається і валідуєтья на беці.
4. Якщо помилка 401 від бека у interceptor зробити рефреш `POST /auth/refresh`.
5. Якщо рефреш фейлиться, редіректити до сторінки логіну `/login`.
6. Якщо юзер вилогінюється, треба почистити токен після запиту логіну і редірект на `/login`.
