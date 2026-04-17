# Frontend Pages

## Roles

| Role           | Scope    | Description                                        |
| -------------- | -------- | -------------------------------------------------- |
| `SYSADMIN`     | Platform | Manages tenants and has global visibility          |
| `SCHOOL_ADMIN` | Tenant   | Manages users, views events and reports for school |
| `TEACHER`      | Tenant   | Manages students, events, lesson plans and reports |
| `STUDENT`      | Tenant   | Participates in events and lesson plans            |
| `JURY`         | Tenant   | Grades competition performances and assigns places |

---

## Auth Pages (public)

### `/login`

**Access:** Everyone (unauthenticated)  
Email + password form. On success, a magic link is sent to the user's email. No session is created here yet.

### `/verify-magic-link?token=...`

**Access:** Everyone (via email link)  
Validates the magic link token. On success, issues `access_token` (in-memory) and `refresh_token` (HttpOnly cookie) and redirects to the appropriate dashboard.

### `/change-password`

**Access:** All authenticated users  
Form to update password. Requires current password and new password confirmation.

---

## SYSADMIN Pages

### `/admin/tenants`

**Access:** `SYSADMIN`  
List of all tenants with name, status (`active` / `suspended` / `archived`) and creation date. Filterable by status.

### `/admin/tenants/new`

**Access:** `SYSADMIN`  
Form to create a new tenant (school). Creates the tenant + initial `SCHOOL_ADMIN` user in one request. Fields: school name, admin full name, admin email, admin password.

### `/admin/tenants/:id`

**Access:** `SYSADMIN`, `SCHOOL_ADMIN`  
Tenant detail view: name, status, settings. `SYSADMIN` can change status (activate / suspend). `SCHOOL_ADMIN` has read-only access to their own tenant.

### `/admin/users`

**Access:** `SYSADMIN`  
Global list of all users across all tenants with filters (by name, email, role, tenant).

---

## SCHOOL_ADMIN Pages

### `/school/users`

**Access:** `SCHOOL_ADMIN`  
List of all users in the school (teachers and students). Shows name, email, role, and membership status.

### `/school/users/new`

**Access:** `SCHOOL_ADMIN`  
Form to create a new user within the tenant. Selects role (`TEACHER` or `STUDENT`). Creates `User`, `Membership`, and `MembershipRole` in one request.

### `/school/dashboard`

**Access:** `SCHOOL_ADMIN`  
Overview: number of teachers, students, upcoming events, and recent reports.

---

## Shared Pages (SCHOOL_ADMIN + TEACHER)

### `/profile/me`

**Access:** All authenticated users  
Displays current user info (`GET /auth/me`): name, email, role, tenant. Entry point for `change-password`.

### `/teachers/:userId`

**Access:** `TEACHER` (own profile), `SCHOOL_ADMIN`  
Teacher profile: full name, email, musical instrument. `TEACHER` can edit own profile; `SCHOOL_ADMIN` can edit any teacher's profile.

### `/students`

**Access:** `TEACHER`, `SCHOOL_ADMIN`  
List of students. `TEACHER` sees only their assigned students. `SCHOOL_ADMIN` sees all students in the tenant.

### `/students/:userId`

**Access:** `TEACHER`, `SCHOOL_ADMIN`  
Student profile: full name, email, musical instrument, class number, assigned lesson plans, event participations.

### `/events`

**Access:** `SCHOOL_ADMIN`, `TEACHER`, `JURY`  
List of events for the tenant (+ global events for `SCHOOL_ADMIN`). Shows name, type, date, status. Filterable by type (webinar / concert / competition).

### `/events/:id`

**Access:** `SCHOOL_ADMIN`, `TEACHER`, `JURY`  
Event detail: description, type, date, participants list link, jury members. `TEACHER` (organizer) can edit. `JURY` can see grading actions.

### `/reports`

**Access:** `SCHOOL_ADMIN`, `TEACHER`  
List of generated reports. `TEACHER` sees reports they created. `SCHOOL_ADMIN` sees all reports for the tenant.

### `/reports/:id`

**Access:** `SCHOOL_ADMIN`, `TEACHER`  
Report detail: aggregated grades and summaries from events and lesson plans. `SCHOOL_ADMIN` can patch (approve/reject) the report.

---

## TEACHER-only Pages

### `/students/new`

**Access:** `TEACHER`  
Form to create a student. Automatically creates a `TeacherStudentAssignment` linking the new student to the teacher.

### `/events/new`

**Access:** `TEACHER`  
Form to create an event: name, type (webinar / concert / competition), date, description. Optionally add jury members immediately.

### `/events/:id/edit`

**Access:** `TEACHER` (organizer only)  
Edit event details. Same fields as create form.

### `/events/:id/participants`

**Access:** `TEACHER`, `JURY`  
Participant list for the event. `TEACHER` can mark attendance (for webinars). Can trigger video upload request.

### `/lesson-plans`

**Access:** `TEACHER`  
List of lesson plans created by the teacher. Shows title, assigned students, and template flag.

### `/lesson-plans/new`

**Access:** `TEACHER`  
Form to create a lesson plan: title, content, mark as template.

### `/lesson-plans/:id`

**Access:** `TEACHER`  
Lesson plan detail: content, assignments, linked students.

### `/lesson-plans/:id/edit`

**Access:** `TEACHER`  
Edit lesson plan content and assignments.

### `/reports/new`

**Access:** `TEACHER`  
Form to generate a new report by selecting a student and a time range. Aggregates event participation and lesson plan assignment data.

---

## JURY-only Pages

### `/events/:id/grading`

**Access:** `JURY`  
Grading view for a competition event. Lists participants with fields to assign place and grade performance. Actions call `POST /events/assign-place` and `POST /events/grade-performance`.

---

## Page–Role Access Matrix

| Page                       | SYSADMIN | SCHOOL_ADMIN | TEACHER | JURY |
| -------------------------- | :------: | :----------: | :-----: | :--: |
| `/login`                   |    ✓     |      ✓       |    ✓    |  ✓   |
| `/verify-magic-link`       |    ✓     |      ✓       |    ✓    |  ✓   |
| `/change-password`         |    ✓     |      ✓       |    ✓    |  ✓   |
| `/profile/me`              |    ✓     |      ✓       |    ✓    |  ✓   |
| `/admin/tenants`           |    ✓     |              |         |      |
| `/admin/tenants/new`       |    ✓     |              |         |      |
| `/admin/tenants/:id`       |    ✓     |   ✓ (own)    |         |      |
| `/admin/users`             |    ✓     |              |         |      |
| `/school/dashboard`        |          |      ✓       |         |      |
| `/school/users`            |          |      ✓       |         |      |
| `/school/users/new`        |          |      ✓       |         |      |
| `/teachers/:userId`        |          |      ✓       | ✓ (own) |      |
| `/students`                |          |      ✓       |    ✓    |      |
| `/students/new`            |          |              |    ✓    |      |
| `/students/:userId`        |          |      ✓       |    ✓    |      |
| `/events`                  |          |      ✓       |    ✓    |  ✓   |
| `/events/new`              |          |              |    ✓    |      |
| `/events/:id`              |          |      ✓       |    ✓    |  ✓   |
| `/events/:id/edit`         |          |              |    ✓    |      |
| `/events/:id/participants` |          |              |    ✓    |  ✓   |
| `/events/:id/grading`      |          |              |         |  ✓   |
| `/lesson-plans`            |          |              |    ✓    |      |
| `/lesson-plans/new`        |          |              |    ✓    |      |
| `/lesson-plans/:id`        |          |              |    ✓    |      |
| `/lesson-plans/:id/edit`   |          |              |    ✓    |      |
| `/reports`                 |          |      ✓       |    ✓    |      |
| `/reports/new`             |          |              |    ✓    |      |
| `/reports/:id`             |          |      ✓       |    ✓    |      |
