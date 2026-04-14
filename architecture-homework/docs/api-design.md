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

- `POST /auth/login` - password validation + request magic link
- `POST /auth/verify-magic-link` - real login
- `POST /auth/refresh`
- `POST /auth/logout`
- `GET /auth/me`

### Identity and Membership

- `GET /users/:tenantId`
- `POST /users/:tenantId` - create user, memebership and memebershipRole in one go

### Teacher

- `GET /teachers/:userId` - teacher or schooladmin can fetch this data
- `PATCH /teachers/:userId` - teacher or shooladmin can edit

#### Student

- `POST /students` - teacher creates a student and they're automatically linked in teacher_student_assignment
- `GET /students/:userId` - teacher or schooladmin can fetch this data
- `PATCH /students/:userId` - teacher or shooladmin can edit

### Events

- `GET /events`
- `GET /events/:id`
- `POST /events` - create an event
- `PATCH /events/:id` - update an event (organizer only)
- `POST /events/register`
- `GET /events/:id/participants`
- `POST /events/:id/attendance` - mark attendance (called for webinars)
- `POST /events/request-video-upload` - (for concerts and webinars) generate presign url where frontend will be able to load the video
- `POST /events/request-video-submission` - the same but for concerts and competitions
- `POST /events/complete-video-upload` - frontend sends video url back to backend where backend stores the video link in the db
- `POST /events/complete-video-submit` - the same but for competitions and concerts
- `POST /events/assign-place` - for competitions
- `POST /events/grade-performance` - for competitions

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

## Some most important endpoints breakdown

## TENANT MANAGEMENT

For all endpoints access rules are the same:

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
    "email": "admin@school7.edu.ua",
    "password": "StrongPass!123"
  }
}
```

Response `201`:

```json
{
  "id": "9c926e8a-2e4e-4490-98a0-725d32ba1628",
  "name": "Kyiv Music School #7",
  "status": "active",
  "createdAt": "2026-04-14T08:10:00Z"
}
```

Internal flow:

1. `TenantManagementModule.SchoolService` validates caller is Sysadmin.
2. Create row in `tenant` (`status=active`).
3. Hash `initialAdmin.password` using strong password hashing algorithm (`argon2id` or `bcrypt`).
4. Create `user` row from `initialAdmin.fullName` + `initialAdmin.email`, storing `passwordHash`.
5. Create `membership` row linking new admin user to the new tenant.
6. Create `membership_role` with `schoolAdmin`.
7. Trigger magic-link onboarding/login email for `initialAdmin.email` (non-blocking side effect).
8. Return data. Membership and role assignment are active immediately, regardless of whether onboarding email is opened/completed as the school admin is now able to login at any time.

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

## AUTH

Shared token handling:

- Browser clients receive both `access_token` and `refresh_token` via `Set-Cookie` headers.
- `access_token`: short-lived JWT, `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`.
- `refresh_token`: long-lived token, `HttpOnly`, `Secure`, `SameSite=Strict`, `Path=/auth`.
- Protected endpoints may also accept `Authorization: Bearer <accessToken>` for non-browser clients.
- Access JWTs are normally validated by signature and expiry; if immediate logout or forced revocation is required, server also checks token `jti` against a revoked-token denylist.
- Refresh tokens are stored server-side only as hashed values and are rotated on refresh.
- Magic-link tokens are stored server-side only as hashed values in the `magic_link` table.

Additional safety measures:

- short access-token TTL
- refresh-token rotation on every refresh
- revoke all refresh sessions on password change or admin suspension
- CSRF protection for cookie-based browser flows
- rate limiting for `/auth/login`, `/auth/request-magic-link`, and `/auth/verify-magic-link`
- never log raw access tokens, refresh tokens, or magic-link tokens

### POST /auth/login

Starts passwordless login by sending magic link.

Access rules:

- Required auth: no

Request body:

```json
{
  "email": "teacher@school-a.edu.ua",
  "password": "Secret9c926e8a"
}
```

Response `200`:

```json
{
  "message": "If account exists, verification link has been sent"
}
```

Internal flow:

1. `AuthService` normalizes email.
2. Check if user exists with the email. If not, return 401.
3. Generate one-time short-lived magic-link token.
4. Store only token hash in `magic_link` table with expiry.
5. Build magic link containing token.
6. Send email via notification provider.
7. Return generic response to avoid account enumeration.

### POST /auth/verify-magic-link

Access rules:

- Required auth: no

Request body:

```json
{
  "token": "<magic-link-token>"
}
```

Response `200`:

Headers:

- `Set-Cookie: access_token=<jwt>; HttpOnly; Secure; SameSite=Lax; Path=/`
- `Set-Cookie: refresh_token=<refresh-token>; HttpOnly; Secure; SameSite=Strict; Path=/auth`

```json
{
  "user": {
    "id": "a40ba58a-1b43-4f6b-8bd3-7dc6d8a47902",
    "fullName": "Olena Bondarenko",
    "email": "teacher@school-a.edu.ua",
    "globalRole": "Member"
  }
}
```

Internal flow:

1. `TokenService` validates token against stored `magic_link.tokenHash` and expiry.
2. Check if magic link already consumed. If consumed, return 401.
3. Mark magic token as consumed.
4. Load `user` and `membership`.
5. Generate short-lived access JWT and refresh token.
6. Persist only refresh-token hash in server-side session storage.
7. Return tokens in cookies and base user data in response body.

### POST /auth/refresh

Issues a new access token and rotates refresh token.

Access rules:

- Required auth: only refresh token in cookie

Request body: none

Response `200`:

Headers:

- `Set-Cookie: access_token=<jwt>; HttpOnly; Secure; SameSite=Lax; Path=/`
- `Set-Cookie: refresh_token=<new-refresh-token>; HttpOnly; Secure; SameSite=Strict; Path=/auth`

```json
{
  "refreshed": true
}
```

Internal flow:

1. Read `refresh_token` cookie.
2. Verify token signature or value and locate matching hashed refresh-token record.
3. Reject expired, revoked, or reused refresh tokens.
4. Rotate refresh token: revoke old one and persist hash of new one.
5. Generate new access JWT.
6. Return new tokens in cookies.

### POST /auth/logout

Acess rules:

- Required auth: yes
- Allowed roles: any authenticated user

Response `204` (no body)

Headers:

- `Set-Cookie: access_token=; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=0`
- `Set-Cookie: refresh_token=; HttpOnly; Secure; SameSite=Strict; Path=/auth; Max-Age=0`

Internal flow:

1. Authenticate access token.
2. Read current refresh token from cookie and revoke its server-side record.
3. Add current access-token `jti` to short-lived denylist when immediate logout is required.
4. Clear auth cookies.
5. Return `204`.

### GET /auth/me

Access rules:

- Required auth: yes
- Allowed roles: any authenticated user

Response `200`:

```json
{
  "user": {
    "id": "a40ba58a-1b43-4f6b-8bd3-7dc6d8a47902",
    "fullName": "Olena Bondarenko",
    "email": "teacher@school-a.edu.ua",
    "globalRole": "Member"
  }
}
```

Internal flow:

1. Read access token from cookie or `Authorization` header.
2. Decode JWT claims - userId and tenantId.
3. Resolve user and current tenant membership.
4. Return profile data.

## IDENTITY AND MEMBERSHIP

### GET /users/:tenantId

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

### POST /users/:tenantId

Creates a platform user identity.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, `Sysadmin`

Request body:

```json
{
  "fullName": "Maksym Kovalenko",
  "email": "maksym.kovalenko@school-a.edu.ua",
  "globalRole": "Member",
  "membershipRole": "Teacher",
  "password": "Secret2000"
}
```

Response `201`:

```json
{
  "id": "f8f2aef3-8c1d-453f-ab2a-fda1faa8e072",
  "fullName": "Maksym Kovalenko",
  "email": "maksym.kovalenko@school-a.edu.ua",
  "globalRole": "Member",
  "membershipRole": "Teacher",
  "isActive": "true"
}
```

Internal flow:

1. Validate uniqueness by email: if email exists, return "Conflict".
2. Create user + membership + membership role.
3. Send onboarding link to the website. User will be redirected to login and fill in their initial credentials. On the first login, they will also be able to change their password.
4. Return created identity.

## EVENTS

### GET /events

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
- `tenantId`: uuid - required if event is not global

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

1. Get tenantId from jwt.
2. If event is global, proceed to the next step. If not, return 401 if event.tenantId !== user.tenantId.
3. `EventQueryService` composes filters.
4. DB read from `event`.
5. Return paginated result.

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

1. Get tenantId from jwt.
2. If event is global, proceed to the next step. If not, return 401 if event.tenantId !== user.tenantId.
3. `EventQueryService` fetches event by ID under tenant context.
4. Aggregate participation count from `event_participation`.

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
4. Insert row into `event` with organizer IDs and other input data.
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

Internal flow:

1. Authorize organizer/admin (check if real organizer by user.id === event.organizerId)
2. Apply patch and return updated event.

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

1. Compare user.id === event.tenantId if event is tenant-scoped. Otherwise, proceed without this check.
2. `EventCommandService` reads event.
3. Insert into `event_participation`.
4. If competition, insert into `competition_participation` with null grade/place.
5. Return participation data.

### POST /events/:id/attendance

Marks participant attendance (called for webinars directly, for competitions and cocerts participation is marked when places and grades are given).

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin`, organizer

Request body:

```json
{
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "attended": true,
  "notes": "arrived on time"
}
```

Internal flow:

1. Authorize user.
2. Update `attended`, `attendanceMarkedAt`, `notes`.
3. Return updated participation.

### POST /events/request-video-upload

Step 1: request a one-time presigned upload URL for event video.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin` or `teacher` (organizer only)

Request body:

```json
{
  "eventId": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "fileName": "webinar-2026-04-20.mp4",
  "contentType": "video/mp4",
  "fileSizeBytes": 523001002
}
```

Response `200`:

```json
{
  "eventId": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "uploadUrl": "https://s3.eu-west-1.amazonaws.com/...signed...",
  "fileKey": "events/11c96f9b-0d8d-45f2-a26d-5f8e14666ec2/webinar-2026-04-20.mp4",
  "expiresInSeconds": 900
}
```

Internal flow:

1. Authorize organizer.
2. Validate `contentType` and `fileSizeBytes` against upload policy.
3. Backend requests S3 presigned PUT URL for a single upload.
4. Return `uploadUrl` + `fileKey` (frontend uploads file directly to S3).

### POST /events/complete-upload

Step 2: confirm upload completion and persist event video reference.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin` or `teacher` (organizer only)

Request body:

```json
{
  "eventId": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "fileKey": "events/11c96f9b-0d8d-45f2-a26d-5f8e14666ec2/webinar-2026-04-20.mp4",
  "videoUrl": "https://cdn.example.com/events/11c96f9b-0d8d-45f2-a26d-5f8e14666ec2/webinar-2026-04-20.mp4"
}
```

Response `200`:

```json
{
  "eventId": "11c96f9b-0d8d-45f2-a26d-5f8e14666ec2",
  "videoUrl": "https://cdn.example.com/events/11c96f9b-0d8d-45f2-a26d-5f8e14666ec2/webinar-2026-04-20.mp4",
  "updatedAt": "2026-04-20T17:00:00Z"
}
```

Internal flow:

1. Frontend uploads file directly to S3 using presigned `uploadUrl` from step 1.
2. Frontend calls this endpoint to confirm upload completion.
3. Backend verifies object existence and ownership.
4. Update `event.videoUrl`.
5. Return updated event video metadata.

### POST /events/assign-place

Assigns place to participant.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin` or `teacher` (jury/organizer roles)

Request body:

```json
{
  "eventId": "4f2d6ce7-8d42-4d7e-8f4a-4e953d3f66b5",
  "participantUserId": "7777d9fb-8c37-4b12-b0b3-09f66dc34f7f",
  "place": 1,
  "juryNotes": "Excellent interpretation and technique"
}
```

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

1. Find event + participation by `(eventId, participantUserId)`.
2. Check if event type is competition.
3. Update `competition_participation.place` and `juryNotes`.
4. Return current scoring state.

### POST /events/grade-performance

Competition only. Assigns numeric grade to participant performance.

Auth and roles:

- Required auth: yes
- Allowed roles: `schoolAdmin` or `teacher` (jury/organizer roles)

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
