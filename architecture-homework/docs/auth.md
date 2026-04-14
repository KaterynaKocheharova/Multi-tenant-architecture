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
