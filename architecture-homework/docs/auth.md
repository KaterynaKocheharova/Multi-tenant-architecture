# 🔐 Аутентифікація

## Флоу

1. Користувач проходить автентифікацію через magic link.
2. Сервер видає:
   - `access_token` (короткоживучий JWT)
   - `refresh_token` (довгоживучий токен сесії)
3. Токени передаються:
   - `access_token` => через `Authorization` header (Bearer token)
   - `refresh_token` => через `Set-Cookie` (HttpOnly cookie)

4. Клієнт:
   - зберігає `access_token` у памʼяті (in-memory)
   - не має доступу до `refresh_token` (HttpOnly cookie)

5. При кожному API запиті:

## Валідація запитів через мідлвару

1. Middleware перевіряє `access_token`:

- перевірка підпису JWT
- перевірка терміну дії

2. Якщо токен валідний:

- додається `user` до `request`
- запит продовжується

<!-- !!!!!!!!!!!!!!!!!!!!!!1 -->

## 🔄 Флоу рефрешу (оновлення access_token)

1. Якщо `access_token` протермінований або API повертає `401 Unauthorized`:

- клієнт викликає:
  ```
  POST /auth/refresh
  ```

2. Браузер автоматично додає `refresh_token` cookie (лише для `/auth/refresh`).

3. Сервер:

- дістає refresh token з cookie
- хешує його і шукає в базі
- перевіряє:
  - чи токен існує
  - чи активна сесія
  - чи не відкликаний токен
- якщо все валідно:
  - генерує новий `access_token`
  - генерує новий `refresh_token` (**rotation**)
  - інвалідовує старий refresh token

4. Відповідь сервера:

- `access_token` → JSON response
- `refresh_token` → новий `Set-Cookie`

5. Клієнт:

- оновлює `access_token` у памʼяті
- повторює початковий запит

---

## 🚪 Logout

1. Клієнт викликає `/auth/logout`
2. Сервер:

- очищає cookie з `refresh_token`
- інвалідовує сесію в базі даних
- (опціонально) додає `jti` access token у denylist

---

## ✉️ Magic Link

- Токени magic link:
- зберігаються тільки як **хеші**
- зберігаються у таблиці `magic_link`
- ніколи не зберігаються у відкритому вигляді

---

## 🌐 Робота клієнта (React SPA)

- `access_token`:
- зберігається в памʼяті (in-memory)
- додається вручну в `Authorization` header

- `refresh_token`:
- недоступний JavaScript
- автоматично надсилається браузером тільки на `/auth/refresh`

---

## 🔒 Додаткові заходи безпеки

- короткий TTL для `access_token`
- refresh token rotation
- інвалідація сесій при:
- зміні пароля
- блокуванні акаунта
- CSRF-захист для `/auth/refresh`
- rate limiting для auth endpointів:
- login
- magic link request
- magic link verify
- заборона логування токенів:
- access_token
- refresh_token
- magic link tokens

---

## 🤔 Чому це безпечно

- **Короткий access token**
  → мінімізує шкоду при компрометації

- **In-memory storage**
  → зменшує ризик XSS-персистентності

- **HttpOnly refresh cookie**
  → недоступний для JavaScript

- **Розділення токенів**
  → компрометація одного не ламає систему

- **Rotation refresh token**
  → викрадений токен швидко стає недійсним

- **Хешування в БД**
  → витік бази не розкриває токени

- **CSRF захист**
  → захищає refresh endpoint

- **Denylist (`jti`)**
  → дозволяє миттєве відкликання сесій

### Налаштування cookie:

- `HttpOnly`
- `Secure`
- `SameSite=Strict`
- `Path=/auth`

### На сервері:

- зберігається **лише хеш токена**
- поля в БД:
  - `issuedAt`
  - `name` (ідентифікатор сесії / пристрою)

---

## 🔄 Оновлення токенів (Refresh Flow)

1. Клієнт викликає `/auth/refresh`
2. Браузер автоматично додає `refresh_token` cookie
3. Сервер:
   - перевіряє хеш токена в БД
   - перевіряє валідність сесії
   - якщо все ок:
     - видає новий `access_token`
     - видає новий `refresh_token`
     - інвалідовує старий refresh token (**rotation**)
4. Відповідь:
   - `access_token` → у JSON
   - `refresh_token` → через cookie

---

## 🚪 Вихід із системи (Logout)

1. Клієнт викликає `/auth/logout`
2. Сервер:
   - очищає cookie з refresh token
   - інвалідовує токен у базі даних (за `name` або session id)

---

## 🛡️ Middleware

### Middleware для Access Token

- Читає токен з:

Authorization: Bearer <access_token>

- Перевіряє:
  - підпис JWT
  - термін дії

- Опціонально:
  - перевірка `jti` у denylist (для миттєвого відкликання)

---

### Middleware для Refresh Token

- Використовується тільки на `/auth/refresh`
- Перевіряє:
  - наявність cookie
  - хеш у базі даних
  - статус сесії

---

## 🌐 Використання на клієнті (React SPA)

### Access Token

- зберігається в памʼяті (не localStorage)
- додається вручну до API запитів

### Refresh Token

- недоступний у JavaScript
- автоматично надсилається браузером через cookie

---

## ✉️ Magic Link

- токени magic link:
  - зберігаються **лише як хеші**
  - знаходяться в таблиці `magic_link`
- ніколи не зберігаються у відкритому вигляді

---

## 🔒 Додаткові заходи безпеки

- короткий час життя `access_token`
- rotation refresh token при кожному оновленні
- інвалідація всіх сесій при:
  - зміні пароля
  - блокуванні акаунта
- CSRF-захист для `/auth/refresh`
- rate limiting для:
  - `/auth/login`
  - `/auth/request-magic-link`
  - `/auth/verify-magic-link`
- заборона логування токенів:
  - access token
  - refresh token
  - magic link токени

---

## 🤔 Чому це безпечно

- **Короткий access token**
  → мінімізує шкоду при викраденні

- **Відсутність localStorage**
  → зменшує ризик XSS-атак

- **HttpOnly refresh cookie**
  → недоступний для JavaScript

- **Розділення токенів**
  → компрометація одного не ламає всю систему

- **Rotation refresh token**
  → викрадений токен швидко стає недійсним

- **Хешування в базі**
  → витік БД не дає можливості використати токени

- **CSRF-захист**
  → захищає refresh endpoint

- **Denylist (`jti`)**
  → дозволяє миттєво завершити сесію

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

<!-- додати обмеження на кожен ендпоїнт по тенант айді +  додаткові + вирішити де їх застосувати -->
