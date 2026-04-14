# API Design

This document defines API contracts for:

- tenant management
- authentication
- identity, memberships, and RBAC management
- events lifecycle and competition operations
- lesson plans and assignments
- report generation and editing

## Full Endpoint Catalog

### Tenant Management

- `POST /tenants`
- `GET /tenants`
- `GET /tenants/:id`
- `PATCH /tenants/:id/status`

### Auth

- `POST /auth/login`
- `POST /auth/verify-magic-link`
- `POST /auth/logout`
- `GET /auth/me`

### Identity and Membership

- `GET /users`
- `POST /users`
- `GET /users/:id`
- `PATCH /users/:id`
- `POST /memberships`
- `PATCH /memberships/:id/status`
- `POST /memberships/:id/roles`
- `DELETE /memberships/:id/roles/:role`

### Teacher Profile

- `PUT /teachers/:userId/profile`

#### Student Profile

- `PUT /students/:userId/profile`

#### Teacher-Student Assignments

- `POST /teacher-student-assignments`
- `PATCH /teacher-student-assignments/:id/status`
- `GET /teacher-student-assignments`

### Events

- `GET /events`
- `GET /events/:id`
- `POST /events`
- `PATCH /events/:id`
- `POST /events/register`
- `GET /events/:id/participants`
- `POST /events/:id/attendance`
- `POST /events/upload-video`
- `POST /events/submit-video`
- `POST /events/assign-place`
- `POST /events/grade-performance`

### Lesson Plans

- `GET /lesson-plans`
- `POST /lesson-plans`
- `GET /lesson-plans/:id`
- `PATCH /lesson-plans/:id`
- `POST /lesson-plans/:id/create-from-template`
- `POST /lesson-plans/:id/assignments`
- `GET /lesson-plans/assignments`
- `PATCH /lesson-plans/assignments/:id`

### Reports

- `POST /reports`
- `GET /reports`
- `GET /reports/:id`
- `PATCH /reports/:id`

## TENANT MANAGEMENT

Shared access rule for all tenant management endpoints in this section:

- Required auth: yes
- Allowed roles: `globalRole = Sysadmin`

### POST /tenants

Creates a new tenant (school).

Request body:

```json
{
  "name": "Kyiv Music School #7",
  "initialAdmin": {
    "fullName": "Olena Bondarenko",
    "email": "admin@school7.edu.ua"
  }
}
```

Response `201`:

```json
{
  "id": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "name": "Kyiv Music School #7",
  "status": "active",
  "createdAt": "2026-04-14T08:10:00Z
}
```

Internal flow:

1. `TenantManagementModule.SchoolService` validates caller is Sysadmin.
2. Create row in `tenant` (`status=active`).
3. Create `user` row from `initialAdmin.fullName` + `initialAdmin.email`.
4. Create `membership` row linking new admin user to the new tenant.
5. Create `membership_role` with `schoolAdmin`.
6. Trigger magic-link onboarding/login email for `initialAdmin.email` (non-blocking side effect).
7. Return data. Membership and role assignment are active immediately, regardless of whether onboarding email is opened/completed as the school admin is now able to login at any time.

### GET /tenants

Query parameters:

- `status` (optional): `active | suspended | archived`
- `search` (optional, by name)
- `page`, `pageSize`

Response `200`:

```json
{
  "items": [
    {
      "id": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
      "name": "Kyiv Music School #7",
      "status": "active",
      "createdAt": "2026-04-14T08:10:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

Internal flow:

1. Validate Sysadmin JWT.
2. Query tenant registry with filters.
3. Return paginated result.

### GET /tenants/:id

Response `200`:

```json
{
  "id": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "name": "Kyiv Music School #7",
  "status": "active",
  "createdAt": "2026-04-14T08:10:00Z",
  "updatedAt": "2026-04-14T08:10:00Z"
}
```

Internal flow:

1. Validate Sysadmin JWT.
2. Load tenant by ID.
3. Return `404` if missing.

### PATCH /tenants/:id/status

Activates, suspends, or archives tenant.

Request body:

```json
{
  "status": "suspended"
}
```

Response `200`:

```json
{
  "id": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "status": "suspended",
  "updatedAt": "2026-04-14T09:00:00Z"
}
```

Internal flow:

1. Validate Sysadmin JWT.
2. Update tenant status.
3. Optionally suspend tenant memberships asynchronously.
4. Return updated tenant state.

<!-- !!!!!!!!!!!!!!!!!1 -->

## AUTH (verification links)

### POST /auth/login

Starts passwordless login by sending magic link.

Auth and roles:

- Required auth: no

Request body:

```json
{
  "email": "teacher@school-a.edu.ua",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628"
}
```

Validation:

- `email`: required, valid email
- `tenantId`: required UUID
- user must exist, be active, and have active membership in tenant

Response `200`:

```json
{
  "message": "If account exists, verification link has been sent"
}
```

Internal flow:

1. `AuthService` normalizes email.
2. Resolve user and tenant membership.
3. Generate one-time short-lived verification token (signed JWT or opaque token with DB/JTI storage).
4. Build link containing token and tenant context.
5. Send email via notification provider.
6. Return generic response to avoid account enumeration.

### POST /auth/verify-magic-link

Verifies magic link token and issues session tokens.

Auth and roles:

- Required auth: no

Request body:

```json
{
  "token": "<magic-link-token>"
}
```

Validation:

- token exists, signature valid, not expired, not already used
- referenced user active
- membership for embedded tenant is active

Response `200`:

```json
{
  "accessToken": "<jwt>",
  "refreshToken": "<refresh-token>",
  "expiresIn": 900,
  "user": {
    "id": "a40ba58a-1b43-4f6b-8bd3-7dc6d8a47902",
    "fullName": "Olena Bondarenko",
    "email": "teacher@school-a.edu.ua",
    "globalRole": "Member"
  },
  "tenant": {
    "id": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
    "roles": ["teacher"]
  }
}
```

Internal flow:

1. `TokenService` validates token integrity and TTL.
2. Mark magic token as consumed (single-use).
3. Load `user`, `membership`, `membership_role`.
4. Build JWT claims (`userId`, `tenantId`, `globalRole`, tenant roles).
5. Return access + refresh tokens.

### POST /auth/logout

Invalidates current session.

Auth and roles:

- Required auth: yes
- Allowed roles: any authenticated user

Request body:

```json
{
  "refreshToken": "<refresh-token>"
}
```

Response `204` (no body)

Internal flow:

1. Authenticate access token.
2. Revoke refresh token (denylist or delete token row).
3. Optionally revoke access-token JTI for immediate logout behavior.
4. Return `204`.

### GET /auth/me

Returns authenticated identity with tenant membership and roles.

Auth and roles:

- Required auth: yes
- Allowed roles: any authenticated user

Response `200`:

```json
{
  "user": {
    "id": "a40ba58a-1b43-4f6b-8bd3-7dc6d8a47902",
    "fullName": "Olena Bondarenko",
    "email": "teacher@school-a.edu.ua",
    "globalRole": "Member",
    "isActive": true
  },
  "membership": {
    "id": "be2e1579-4af1-4bc8-9bc8-e292af5f41c9",
    "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
    "status": "active",
    "roles": ["teacher"]
  }
}
```

Internal flow:

1. Decode JWT claims.
2. Resolve user and current tenant membership.
3. Return consolidated profile DTO for frontend bootstrapping.

## IDENTITY AND MEMBERSHIP

### GET /users

Lists users visible in current tenant.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`

Query parameters:

- `role` (optional): `schoolAdmin | teacher | student`
- `search` (optional by name/email)
- `page`, `pageSize`

Response `200`:

```json
{
  "items": [
    {
      "id": "a40ba58a-1b43-4f6b-8bd3-7dc6d8a47902",
      "fullName": "Olena Bondarenko",
      "email": "teacher@school-a.edu.ua",
      "isActive": true,
      "membership": {
        "id": "be2e1579-4af1-4bc8-9bc8-e292af5f41c9",
        "status": "active",
        "roles": ["teacher"]
      }
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

Internal flow:

1. Open tenant transaction context.
2. Join `user`, `membership`, `membership_role` for caller tenant.
3. Apply filters and pagination.

### POST /users

Creates a platform user identity.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `Sysadmin`

Request body:

```json
{
  "fullName": "Maksym Kovalenko",
  "email": "maksym.kovalenko@school-a.edu.ua",
  "globalRole": "Member"
}
```

Validation:

- unique email (`citext`)
- `globalRole` in `Sysadmin | Member`

Response `201`:

```json
{
  "id": "f8f2aef3-8c1d-453f-ab2a-fda1faa8e072",
  "fullName": "Maksym Kovalenko",
  "email": "maksym.kovalenko@school-a.edu.ua",
  "globalRole": "Member",
  "isActive": true
}
```

Internal flow:

1. Validate uniqueness by email.
2. Insert row in `user`.
3. Return created identity.

### GET /users/:id

Returns user profile and tenant membership projection.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher` (self or assigned students), `student` (self)

Internal flow:

1. Open tenant context.
2. Resolve user and role-dependent visibility checks.
3. Return profile + roles + details (`teacher_details` or `student_details` when present).

### PATCH /users/:id

Updates user identity fields.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin` or self for limited fields

Request body:

```json
{
  "fullName": "Maksym K."
}
```

Internal flow:

1. Validate field-level permissions.
2. Update `user` row.
3. Return updated representation.

### POST /memberships

Links existing user to tenant.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `Sysadmin`

Request body:

```json
{
  "userId": "f8f2aef3-8c1d-453f-ab2a-fda1faa8e072",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "roles": ["teacher"]
}
```

Validation:

- user can have only one membership (current model constraint)
- tenant exists and active
- role list non-empty

Response `201`:

```json
{
  "membershipId": "5006a7a7-970d-4daf-b7ea-366737276637",
  "userId": "f8f2aef3-8c1d-453f-ab2a-fda1faa8e072",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "status": "active",
  "roles": ["teacher"]
}
```

Internal flow:

1. Create `membership` row.
2. Insert one row per role into `membership_role`.
3. Commit atomically.

### PATCH /memberships/:id/status

Suspends or reactivates membership.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `Sysadmin`

Request body:

```json
{
  "status": "suspended"
}
```

Internal flow:

1. Resolve membership in tenant scope.
2. Update `membership.status`.
3. Return status DTO.

### POST /memberships/:id/roles

Adds role to membership.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `Sysadmin`

Request body:

```json
{
  "role": "student"
}
```

Validation:

- unique `(membershipId, role)`

Internal flow:

1. Insert into `membership_role`.
2. Return updated role list.

### DELETE /memberships/:id/roles/:role

Removes role from membership.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `Sysadmin`

Validation:

- cannot remove the last role from active membership

Internal flow:

1. Delete role row.
2. Enforce at least one remaining role.
3. Return updated role list.

## TEACHER AND STUDENT PROFILES

### PUT /teachers/:userId/profile

Creates or updates teacher details.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, self (teacher)

Request body:

```json
{
  "musicalInstrument": "Piano"
}
```

Internal flow:

1. Validate that user has `teacher` membership role.
2. Upsert into `teacher_details`.
3. Return profile DTO.

### PUT /students/:userId/profile

Creates or updates student details.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, assigned teacher, self (student, limited fields)

Request body:

```json
{
  "musicalInstrument": "Violin",
  "classNumber": "6-A"
}
```

Internal flow:

1. Validate user has `student` role.
2. Authorize caller by role or assignment.
3. Upsert into `student_details`.

### POST /teacher-student-assignments

Assigns teacher to student in one tenant.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`

Request body:

```json
{
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "studentUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f"
}
```

Validation:

- unique `(tenantId, teacherUserId, studentUserId)`
- teacher and student roles must match expected role types

Response `201`:

```json
{
  "id": "67f57c55-a35f-4209-b24f-ddbf1fb5fdbd",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "studentUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "status": "active"
}
```

Internal flow:

1. Resolve both users in same tenant.
2. Validate roles and uniqueness.
3. Insert into `teacher_student_assignment`.

### PATCH /teacher-student-assignments/:id/status

Activates/inactivates teacher-student mapping.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`

Request body:

```json
{
  "status": "inactive"
}
```

Internal flow:

1. Resolve assignment by ID with tenant filter.
2. Update status.
3. Return updated assignment DTO.

### GET /teacher-student-assignments

Returns assignment map for teacher dashboards and admin views.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`

Internal flow:

1. `schoolAdmin`: list all assignments in tenant.
2. `teacher`: list assignments where `teacherUserId = caller.userId`.
3. Return list.

## EVENTS

### GET /events

Returns list of visible events.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`, `student`, `Sysadmin`

Query parameters:

- `type`: `webinar | concert | competition` (optional)
- `scope`: `TENANT | GLOBAL` (optional)
- `from`: ISO datetime (optional)
- `to`: ISO datetime (optional)
- `page`: integer, default `1`
- `pageSize`: integer, default `20`, max `100`

Response `200`:

```json
{
  "items": [
    {
      "id": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
      "scope": "GLOBAL",
      "tenantId": null,
      "type": "competition",
      "name": "Spring Piano Challenge",
      "topic": "Junior category",
      "videoUrl": null,
      "organizerUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
      "startDate": "2026-05-01T10:00:00Z",
      "endDate": "2026-05-01T13:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

Internal flow:

1. Resolve JWT claims and open tenant-scoped transaction context.
2. `EventQueryService` composes filters.
3. DB read from `event`; RLS allows own-tenant rows and global rows (`scope=GLOBAL`) according to policy.
4. Return paginated result.

### GET /events/:id

Returns details of one event.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`, `student`, `Sysadmin`

Path params:

- `id` UUID

Response `200`:

```json
{
  "id": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "scope": "GLOBAL",
  "tenantId": null,
  "type": "competition",
  "name": "Spring Piano Challenge",
  "topic": "Junior category",
  "videoUrl": null,
  "startDate": "2026-05-01T10:00:00Z",
  "endDate": "2026-05-01T13:00:00Z",
  "participationCount": 42
}
```

Internal flow:

1. `EventQueryService` fetches event by ID under tenant context.
2. If no visible row after RLS, return `404`.
3. Aggregate participation count from `event_participation`.

### POST /events

Creates an event (`webinar`, `concert`, `competition`).

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`

Request body:

```json
{
  "scope": "TENANT",
  "type": "webinar",
  "name": "Methodology Webinar",
  "topic": "Choir rehearsal techniques",
  "startDate": "2026-04-20T15:00:00Z",
  "endDate": "2026-04-20T16:30:00Z",
  "organizerUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2"
}
```

Validation:

- `endDate >= startDate`
- `scope=TENANT` implies current `tenantId` is attached
- `scope=GLOBAL` allowed only for authorized roles (`schoolAdmin` or Sysadmin depending policy)
- organizer must belong to caller tenant for tenant-scoped events

Response `201`:

```json
{
  "id": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "scope": "TENANT",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "type": "webinar",
  "name": "Methodology Webinar",
  "topic": "Choir rehearsal techniques",
  "videoUrl": null,
  "organizerUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "startDate": "2026-04-20T15:00:00Z",
  "endDate": "2026-04-20T16:30:00Z"
}
```

Internal flow:

1. `EventService` routes to `EventCommandService`.
2. Select strategy by `type` (`WebinarStrategy`, `ConcertStrategy`, `CompetitionStrategy`).
3. Strategy-level validation runs.
4. Insert row into `event` with creator and organizer IDs.
5. Return created event.

### PATCH /events/:id

Updates event metadata.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, event organizer

Request body (partial):

```json
{
  "name": "Updated event name",
  "startDate": "2026-04-20T15:30:00Z",
  "endDate": "2026-04-20T17:00:00Z"
}
```

Validation:

- end date must remain `>=` start date
- event scope/tenant invariants cannot be broken

Internal flow:

1. Resolve event under tenant context.
2. Authorize organizer/admin.
3. Apply patch and return updated event.

### POST /events/register

Registers participant in event.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`, `student`

Request body:

```json
{
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "roleInEvent": "performer",
  "notes": "solo piano"
}
```

Validation:

- unique pair `(eventId, participantUserId)`
- event exists and caller can access it
- if event scope is `TENANT`, participant must belong to same tenant
- if event type is `competition`, create companion row in `competition_participation`

Response `201`:

```json
{
  "id": "b76607e7-e5b1-49bc-8130-836b3d5d1ed7",
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "roleInEvent": "performer",
  "attended": false,
  "notes": "solo piano"
}
```

Internal flow:

1. `EventCommandService` reads event.
2. Enforce RBAC and tenant/global rules.
3. Insert into `event_participation`.
4. If competition, insert into `competition_participation` with null grade/place.
5. Return participation DTO.

### GET /events/:id/participants

Returns participants with competition fields when applicable.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`, `Sysadmin`

Response `200`:

```json
{
  "items": [
    {
      "id": "b76607e7-e5b1-49bc-8130-836b3d5d1ed7",
      "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
      "roleInEvent": "performer",
      "attended": true,
      "grade": 96.5,
      "place": 1
    }
  ]
}
```

Internal flow:

1. Validate caller can view event.
2. Query `event_participation` and left-join `competition_participation`.
3. Return participant list.

### POST /events/:id/attendance

Marks participant attendance.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, organizer, designated teacher

Request body:

```json
{
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "attended": true,
  "notes": "arrived on time"
}
```

Internal flow:

1. Resolve participation row by event and user.
2. Update `attended`, `attendanceMarkedAt`, `notes`.
3. Return updated participation DTO.

### POST /events/upload-video

Organizer uploads final event video link (webinar recording or concert full video).

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher` (organizer only)

Request body:

```json
{
  "eventId": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "videoUrl": "https://cdn.example.com/videos/webinar-2026-04-20.mp4"
}
```

Validation:

- event type must be `webinar` or `concert`
- caller must be event organizer or school admin
- valid URL

Response `200`:

```json
{
  "eventId": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "videoUrl": "https://cdn.example.com/videos/webinar-2026-04-20.mp4",
  "updatedAt": "2026-04-20T17:00:00Z"
}
```

Internal flow:

1. Load event under tenant context.
2. Validate type and caller rights.
3. Update `event.videoUrl`.
4. Return updated metadata.

### POST /events/submit-video

Teacher submits participant video for concert performance or competition.

Auth and roles:

- Required auth: yes
- Allowed roles: `teacher`

Request body:

```json
{
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "videoUrl": "https://cdn.example.com/videos/student-performance.mp4",
  "comment": "regional stage submission"
}
```

Validation:

- event type must be `concert` or `competition`
- participant must already be registered
- caller must be assigned teacher for student or school admin

Response `200`:

```json
{
  "participationId": "b76607e7-e5b1-49bc-8130-836b3d5d1ed7",
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "submission": {
    "videoUrl": "https://cdn.example.com/videos/student-performance.mp4",
    "comment": "regional stage submission"
  }
}
```

Internal flow:

1. Load event + participation + teacher-student assignment.
2. Validate ownership and event type.
3. Store submission in `event_participation.notes` as structured JSON text payload (video URL + comment).
4. Return submission view model.

### POST /events/assign-place

Competition only. Assigns place to participant.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher` (jury/organizer roles by policy)

Request body:

```json
{
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "place": 1,
  "juryNotes": "Excellent interpretation and technique"
}
```

Validation:

- event type must be `competition`
- place positive integer
- participant must be in competition participation set

Response `200`:

```json
{
  "participationId": "b76607e7-e5b1-49bc-8130-836b3d5d1ed7",
  "grade": 96.5,
  "place": 1,
  "juryNotes": "Excellent interpretation and technique"
}
```

Internal flow:

1. Resolve event + participation by `(eventId, participantUserId)`.
2. Ensure competition subtype row exists in `competition_participation`.
3. Update `competition_participation.place` and `juryNotes`.
4. Return current scoring state.

### POST /events/grade-performance

Competition only. Assigns numeric grade to participant performance.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher` (jury/organizer roles by policy)

Request body:

```json
{
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "grade": 96.5,
  "juryNotes": "Strong dynamics, minor rhythm issue"
}
```

Validation:

- event type must be `competition`
- `grade` numeric in accepted range (example `0..100`)

Response `200`:

```json
{
  "participationId": "b76607e7-e5b1-49bc-8130-836b3d5d1ed7",
  "grade": 96.5,
  "place": 1,
  "juryNotes": "Strong dynamics, minor rhythm issue"
}
```

Internal flow:

1. Resolve event and competition participation row.
2. Validate caller has grading rights.
3. Update `competition_participation.grade` and `juryNotes`.
4. Return updated competition scoring snapshot.

## LESSON PLANS

### GET /lesson-plans

Lists lesson plans for tenant.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`

Query parameters:

- `teacherUserId` (optional)
- `isTemplate` (optional)
- `page`, `pageSize`

Internal flow:

1. Open tenant transaction context.
2. Query `lesson_plan` with filters and pagination.
3. Return plan list.

### POST /lesson-plans

Creates lesson plan or template.

Auth and roles:

- Required auth: yes
- Allowed roles: `teacher`, `schoolAdmin`

Request body:

```json
{
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "isTemplate": false,
  "title": "Weekly Piano Practice",
  "content": {
    "goals": ["scales", "sight reading"],
    "durationMinutes": 45
  }
}
```

Validation:

- `content` required JSON object
- teacher can create only own plans unless school admin

Response `201`:

```json
{
  "id": "ed85d3f5-c56f-4b29-9456-9fa8653d4e6f",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "isTemplate": false,
  "title": "Weekly Piano Practice",
  "content": {
    "goals": ["scales", "sight reading"],
    "durationMinutes": 45
  }
}
```

Internal flow:

1. Authorize teacher/admin.
2. Insert into `lesson_plan`.
3. Return created DTO.

### GET /lesson-plans/:id

Returns one lesson plan.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, plan owner teacher

Internal flow:

1. Resolve plan by ID in tenant context.
2. Apply ownership rules.
3. Return full plan.

### PATCH /lesson-plans/:id

Updates title/content/template flag.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, plan owner teacher

Internal flow:

1. Resolve and authorize.
2. Update mutable fields.
3. Return updated plan.

### POST /lesson-plans/:id/create-from-template

Creates a new lesson plan from an existing template plan.

Auth and roles:

- Required auth: yes
- Allowed roles: `teacher`, `schoolAdmin`

Internal flow:

1. Load source plan.
2. Validate source plan has `isTemplate=true`.
3. Copy `title/content` into a new row with `isTemplate=false` and caller/target teacher assignment.
4. Insert new row and return created plan metadata.

### POST /lesson-plans/:id/assignments

Assigns plan to student by date.

Auth and roles:

- Required auth: yes
- Allowed roles: `teacher`, `schoolAdmin`

Request body:

```json
{
  "studentUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "assignedDate": "2026-04-22",
  "notes": "focus on articulation"
}
```

Validation:

- unique `(lessonPlanId, studentUserId, assignedDate)`
- assignment must respect teacher-student mapping (or school admin override)

Response `201`:

```json
{
  "id": "69f67f65-f6f1-4f84-8a8a-3ff6e2e0e4b7",
  "lessonPlanId": "ed85d3f5-c56f-4b29-9456-9fa8653d4e6f",
  "studentUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "assignedDate": "2026-04-22",
  "status": "planned",
  "notes": "focus on articulation"
}
```

Internal flow:

1. Validate plan visibility and teacher-student mapping.
2. Insert into `lesson_plan_assignment`.
3. Return created assignment.

### GET /lesson-plans/assignments

Lists lesson plan assignments.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`, `student`

Query parameters:

- `studentUserId` (optional)
- `teacherUserId` (optional)
- `status` (optional)
- `fromDate`, `toDate` (optional)

Internal flow:

1. Build caller-scope filter:
   - admin: tenant-wide
   - teacher: own students
   - student: self only
2. Query `lesson_plan_assignment` joined with plan metadata.
3. Return paged list.

### PATCH /lesson-plans/assignments/:id

Updates assignment status and notes.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, assigned teacher, student (status to completed only)

Request body:

```json
{
  "status": "completed",
  "notes": "performed with good tempo"
}
```

Validation:

- status in `planned | completed | cancelled`

Internal flow:

1. Resolve assignment with tenant + visibility checks.
2. Apply role-aware field restrictions.
3. Update and return assignment.

## REPORTS

### POST /reports

Generates report snapshot and stores it.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`

Request body:

```json
{
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "periodStart": "2026-01-01",
  "periodEnd": "2026-05-31",
  "reportType": "semester",
  "filters": {
    "includeEvents": true,
    "includeLessons": true,
    "eventTypes": ["webinar", "concert", "competition"]
  }
}
```

Validation:

- `periodEnd >= periodStart`
- `teacherUserId` optional for school admin, required for teacher self-report
- `reportType` in: `annual | semester | term | random`

Response `201`:

```json
{
  "id": "af36e7bd-9633-484e-8eb6-5b064a9e4dff",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "requestedByUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "periodStart": "2026-01-01",
  "periodEnd": "2026-05-31",
  "reportType": "semester",
  "generatedAt": "2026-06-01T12:00:00Z",
  "snapshotPayload": {
    "summary": {
      "eventsCount": 14,
      "lessonsCompleted": 96,
      "competitionAwards": 5
    }
  }
}
```

Internal flow:

1. `ReportService` chooses generator strategy by request filter (`EventReportGenerator`, `AcademicReportGenerator`, or mixed).
2. Read source rows from `event_participation`, `competition_participation`, `lesson_plan_assignment` within tenant context.
3. Compute aggregate metrics and narrative blocks.
4. Persist immutable snapshot to `report.snapshotPayload`.
5. Return saved report.

### PATCH /reports/:id

Edits a saved report snapshot.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher` (owner only)

Path params:

- `id` UUID

Request body:

```json
{
  "snapshotPayload": {
    "summary": {
      "teacherComment": "Updated after final jury confirmations"
    }
  }
}
```

Validation:

- report exists in caller tenant
- teachers can edit only reports they requested; school admin can edit any tenant report

Response `200`:

```json
{
  "id": "af36e7bd-9633-484e-8eb6-5b064a9e4dff",
  "updated": true,
  "snapshotPayload": {
    "summary": {
      "teacherComment": "Updated after final jury confirmations"
    }
  }
}
```

Internal flow:

1. Fetch report under tenant context (RLS).
2. Check ownership/role rule.
3. Merge or replace `snapshotPayload` according to patch semantics.
4. Save and return updated payload.

### GET /reports

Returns reports visible to caller. Primary use: school admin reporting dashboard.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `teacher`

Query parameters:

- `teacherUserId` (optional, school admin only)
- `reportType` (optional)
- `periodStartFrom` (optional)
- `periodEndTo` (optional)
- `page`, `pageSize`

Response `200`:

```json
{
  "items": [
    {
      "id": "af36e7bd-9633-484e-8eb6-5b064a9e4dff",
      "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
      "reportType": "semester",
      "periodStart": "2026-01-01",
      "periodEnd": "2026-05-31",
      "generatedAt": "2026-06-01T12:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

Internal flow:

1. Resolve tenant context from JWT.
2. `ReportService` applies caller-scope filters:
   - school admin: tenant-wide
   - teacher: own reports only
3. Query `report` table with pagination.
4. Return compact report list.

### GET /reports/:id

Returns full report payload (`snapshotPayload`) for one report.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, report owner teacher

Response `200`:

```json
{
  "id": "af36e7bd-9633-484e-8eb6-5b064a9e4dff",
  "tenantId": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "teacherUserId": "9bdf0506-cb0f-4f54-860e-a6eb4f742fc2",
  "periodStart": "2026-01-01",
  "periodEnd": "2026-05-31",
  "reportType": "semester",
  "snapshotPayload": {
    "summary": {
      "eventsCount": 14,
      "lessonsCompleted": 96
    }
  },
  "generatedAt": "2026-06-01T12:00:00Z"
}
```

Internal flow:

1. Resolve report by ID inside tenant RLS context.
2. Enforce owner/admin visibility.
3. Return report document.

## Notes On Endpoint Normalization

- Original starter listed `GET /events` twice. In this document second read endpoint is normalized to `GET /events/:id`.
- Endpoint names remain aligned with the provided starter for implementation traceability.

## Table To Endpoint Coverage

This section shows where each ERD table is created/read/updated by the API.

- `tenant`: `POST /tenants`, `GET /tenants`, `GET /tenants/:id`, `PATCH /tenants/:id/status`
- `user`: `POST /users`, `POST /tenants` (creates initial admin), `GET /users`, `GET /users/:id`, `PATCH /users/:id`, `GET /auth/me`
- `teacher_details`: `PUT /teachers/:userId/profile`
- `student_details`: `PUT /students/:userId/profile`
- `membership`: `POST /memberships`, `PATCH /memberships/:id/status`, `GET /auth/me`, tenant-scoped user listing
- `membership_role`: `POST /memberships/:id/roles`, `DELETE /memberships/:id/roles/:role`, role expansion in `GET /users` and `GET /auth/me`
- `teacher_student_assignment`: `POST /teacher-student-assignments`, `PATCH /teacher-student-assignments/:id/status`, `GET /teacher-student-assignments`
- `event`: `POST /events`, `PATCH /events/:id`, `GET /events`, `GET /events/:id`, `POST /events/upload-video`
- `event_participation`: `POST /events/register`, `GET /events/:id/participants`, `POST /events/:id/attendance`, `POST /events/submit-video`
- `competition_participation`: `POST /events/register` (create for competition), `POST /events/grade-performance`, `POST /events/assign-place`, `GET /events/:id/participants`
- `award`: not exposed as API endpoint in current scope (table reserved for future certificate workflows)
- `lesson_plan`: `POST /lesson-plans`, `GET /lesson-plans`, `GET /lesson-plans/:id`, `PATCH /lesson-plans/:id`, `POST /lesson-plans/:id/create-from-template`
- `lesson_plan_assignment`: `POST /lesson-plans/:id/assignments`, `GET /lesson-plans/assignments`, `PATCH /lesson-plans/assignments/:id`
- `report`: `POST /reports`, `GET /reports`, `GET /reports/:id`, `PATCH /reports/:id`
