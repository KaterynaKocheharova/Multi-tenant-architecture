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

## Доступ по ролях

```mermaid
flowchart LR
   %% Roles
   R_SYS[SYSADMIN]
   R_SA[SCHOOL_ADMIN]
   R_T[TEACHER]
   R_J[JURY]

   %% Public / all authenticated
   P_LOGIN[/login]
   P_VERIFY[/verify-magic-link]
   P_CH_PASS[/change-password]
   P_PROFILE[/profile/me]

   %% SYSADMIN only
   S_TENANTS[/admin/tenants]
   S_TENANTS_NEW[/admin/tenants/new]
   S_USERS[/admin/users]

   %% SYSADMIN + SCHOOL_ADMIN
   SA_TENANT_ID[/admin/tenants/:id]

   %% SCHOOL_ADMIN only
   A_DASH[/school/dashboard]
   A_USERS[/school/users]
   A_USERS_NEW[/school/users/new]

   %% SCHOOL_ADMIN + TEACHER
   AT_TEACHER_ID[/teachers/:userId]
   AT_STUDENTS[/students]
   AT_STUDENT_ID[/students/:userId]
   AT_REPORTS[/reports]
   AT_REPORT_ID[/reports/:id]

   %% TEACHER only
   T_STUDENT_NEW[/students/new]
   T_EVENT_NEW[/events/new]
   T_EVENT_EDIT[/events/:id/edit]
   T_LP[/lesson-plans]
   T_LP_NEW[/lesson-plans/new]
   T_LP_ID[/lesson-plans/:id]
   T_LP_EDIT[/lesson-plans/:id/edit]
   T_REPORT_NEW[/reports/new]

   %% SCHOOL_ADMIN + TEACHER + JURY
   E_EVENTS[/events]
   E_EVENT_ID[/events/:id]

   %% TEACHER + JURY
   EJ_PART[/events/:id/participants]

   %% JURY only
   J_GRADING[/events/:id/grading]

   %% All roles
   R_SYS --> P_LOGIN
   R_SA --> P_LOGIN
   R_T --> P_LOGIN
   R_J --> P_LOGIN

   R_SYS --> P_VERIFY
   R_SA --> P_VERIFY
   R_T --> P_VERIFY
   R_J --> P_VERIFY

   R_SYS --> P_CH_PASS
   R_SA --> P_CH_PASS
   R_T --> P_CH_PASS
   R_J --> P_CH_PASS

   R_SYS --> P_PROFILE
   R_SA --> P_PROFILE
   R_T --> P_PROFILE
   R_J --> P_PROFILE

   %% SYSADMIN
   R_SYS --> S_TENANTS
   R_SYS --> S_TENANTS_NEW
   R_SYS --> S_USERS

   %% SYSADMIN + SCHOOL_ADMIN
   R_SYS --> SA_TENANT_ID
   R_SA --> SA_TENANT_ID

   %% SCHOOL_ADMIN
   R_SA --> A_DASH
   R_SA --> A_USERS
   R_SA --> A_USERS_NEW

   %% SCHOOL_ADMIN + TEACHER
   R_SA --> AT_TEACHER_ID
   R_T --> AT_TEACHER_ID
   R_SA --> AT_STUDENTS
   R_T --> AT_STUDENTS
   R_SA --> AT_STUDENT_ID
   R_T --> AT_STUDENT_ID
   R_SA --> AT_REPORTS
   R_T --> AT_REPORTS
   R_SA --> AT_REPORT_ID
   R_T --> AT_REPORT_ID

   %% TEACHER
   R_T --> T_STUDENT_NEW
   R_T --> T_EVENT_NEW
   R_T --> T_EVENT_EDIT
   R_T --> T_LP
   R_T --> T_LP_NEW
   R_T --> T_LP_ID
   R_T --> T_LP_EDIT
   R_T --> T_REPORT_NEW

   %% SCHOOL_ADMIN + TEACHER + JURY
   R_SA --> E_EVENTS
   R_T --> E_EVENTS
   R_J --> E_EVENTS
   R_SA --> E_EVENT_ID
   R_T --> E_EVENT_ID
   R_J --> E_EVENT_ID

   %% TEACHER + JURY
   R_T --> EJ_PART
   R_J --> EJ_PART

   %% JURY
   R_J --> J_GRADING
```
