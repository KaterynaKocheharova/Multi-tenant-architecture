# Authorization Matrix (Canonical)

This file is the single source of truth for API authorization.

Rules:

- RBAC: role must be in the `Allowed roles` set.
- ABAC: all listed checks must pass.
- RLS: database-level protection is mandatory but secondary.

## Roles

- `SYSADMIN`
- `SCHOOL_ADMIN`
- `TEACHER`
- `STUDENT`
- `JURY`

## Endpoint Matrix

| Endpoint                                      | Allowed roles                     | ABAC checks                                |
| --------------------------------------------- | --------------------------------- | ------------------------------------------ |
| `POST /tenants`                               | `SYSADMIN`                        | -                                          |
| `GET /tenants`                                | `SYSADMIN`                        | -                                          |
| `GET /tenants/:id`                            | `SYSADMIN`, `SCHOOL_ADMIN`        | `isSelfTenantOrSysadmin`                   |
| `PATCH /tenants/:id/status`                   | `SYSADMIN`                        | -                                          |
| `POST /auth/login`                            | Public                            | -                                          |
| `POST /auth/verify-magic-link`                | Public                            | -                                          |
| `POST /auth/refresh`                          | Authenticated                     | `hasActiveSession`                         |
| `POST /auth/logout`                           | Authenticated                     | `hasActiveSession`                         |
| `GET /auth/me`                                | Authenticated                     | -                                          |
| `POST /auth/change-password`                  | Authenticated                     | `isOwner`                                  |
| `GET /users/all`                              | `SYSADMIN`                        | -                                          |
| `GET /users`                                  | `SCHOOL_ADMIN`                    | `isSameTenant`                             |
| `POST /users`                                 | `SCHOOL_ADMIN`                    | `isSameTenant`                             |
| `GET /teachers/:userId`                       | `TEACHER`, `SCHOOL_ADMIN`         | `isSameTenant`                             |
| `PATCH /teachers/:userId`                     | `TEACHER`, `SCHOOL_ADMIN`         | `isSameTenant` and `isOwnerOrSchoolAdmin`  |
| `POST /students`                              | `TEACHER`                         | `isSameTenant`                             |
| `GET /students/:userId`                       | `TEACHER`, `SCHOOL_ADMIN`         | `isAssignedTeacherOrSchoolAdminSameTenant` |
| `PATCH /students/:userId`                     | `TEACHER`                         | `isAssignedTeacher`                        |
| `GET /events`                                 | `SCHOOL_ADMIN`, `TEACHER`, `JURY` | `canViewGlobalOrTenantEvent`               |
| `GET /events/:id`                             | `SCHOOL_ADMIN`, `TEACHER`, `JURY` | `canViewGlobalOrTenantEvent`               |
| `POST /events`                                | `TEACHER`                         | `isSameTenant`                             |
| `PATCH /events/:id`                           | `TEACHER`                         | `isEventOrganizer`                         |
| `POST /events/add-jury-member`                | `TEACHER`                         | `isEventOrganizer`                         |
| `POST /events/register`                       | `TEACHER`                         | `canRegisterStudentForEvent`               |
| `GET /events/:id/participants`                | `TEACHER`, `JURY`                 | `isEventOrganizerOrEventJury`              |
| `POST /events/:id/attendance`                 | `TEACHER`                         | `isEventOrganizer`                         |
| `POST /events/request-video-upload`           | `TEACHER`                         | `isEventOrganizer`                         |
| `POST /events/complete-upload`                | `TEACHER`                         | `isEventOrganizer`                         |
| `POST /events/assign-place`                   | `JURY`                            | `isEventJury`                              |
| `POST /events/grade-performance`              | `JURY`                            | `isEventJury`                              |
| `GET /lesson-plans`                           | `TEACHER`                         | `isSameTenant`                             |
| `POST /lesson-plans`                          | `TEACHER`                         | `isSameTenant`                             |
| `GET /lesson-plans/:id`                       | `TEACHER`                         | `isSameTenant`                             |
| `PATCH /lesson-plans/:id`                     | `TEACHER`                         | `isOwner`                                  |
| `POST /lesson-plans/:id/create-from-template` | `TEACHER`                         | `isOwner`                                  |
| `POST /lesson-plans/:id/assignments`          | `TEACHER`                         | `isAssignedTeacher`                        |
| `POST /reports`                               | `TEACHER`                         | `isOwner`                                  |
| `GET /reports`                                | `SCHOOL_ADMIN`, `TEACHER`         | `isOwnerOrSchoolAdminSameTenant`           |
| `GET /reports/:id`                            | `SCHOOL_ADMIN`, `TEACHER`         | `isOwnerOrSchoolAdminSameTenant`           |
| `PATCH /reports/:id`                          | `SCHOOL_ADMIN`                    | `isSameTenant`                             |

## Synchronization Rule

- `docs/rbac.md` is a human-friendly role overview derived from this file.
- `docs/api-design.md` endpoint behavior must reference this file for authorization.
- If conflict appears, this file wins.
