```http
```
```ts
```
```http
```
<!-- DOCS_NAV_START -->
[Docs Home](README.md) | [API Design](api-design.md) | [Auth](auth.md) | [RBAC](rbac.md) | [Data Model](data-model.md) | [Security](security.md) | [Deployment](deployment.md) | [Containers](containers.md) | [Context](context.md) | [Frontend](front.md) | [NFR](nfr.md) | [Req-Res Propagation](req-res-propagation.md) | [Risks](risks.md)
<!-- DOCS_NAV_END -->

## Навігація в документі

- [Флоу](#флоу)
  - [Налаштування refresh cookie](#налаштування-refresh-cookie)
  - [На сервері:](#на-сервері)
- [Валідація запитів через мідлвару](#валідація-запитів-через-мідлвару)
- [Флоу рефрешу](#флоу-рефрешу)
- [Logout](#logout)

<!-- DOCS_TOC_START -->
<!-- DOCS_TOC_END -->


# Аутентифікація

## Флоу

1. Користувач проходить автентифікацію через magic link.
2. Сервер видає:
   - `access_token` (короткоживучий JWT) - має `userId`,`jti` - ідентифікатор токена.
   - `refresh_token` cookie (довгоживучий токен сесії)

### Налаштування refresh cookie

- `HttpOnly`
- `Secure`
- `SameSite=Lax`
- `Path=/auth/refresh`

Приклад:

Set-Cookie: refresh_token=<opaque>; HttpOnly; Secure; SameSite=Lax; Path=/auth/refresh

### На сервері:

- зберігається **лише хеш токена**

{
  id: uuid,
  userId: uuid,
  tokenHash: string,
  issuedAt: Date,
  revoked: boolean
}

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

## Флоу рефрешу

1. Якщо API повертає `401 Unauthorized`:

- клієнт викликає:

POST /auth/refresh

2. Браузер автоматично додає `refresh_token` cookie.

3. Сервер:

- дістає refresh token з cookie
- хешує його і шукає в базі
- перевіряє:
- чи токен існує
- чи активна сесія
- якщо все ок:
- генерує новий `access_token`
- генерує новий `refresh_token`
- інвалідовує старий refresh token

4. Відповідь сервера:

- `access_token` → JSON response
- `refresh_token` → новий `Set-Cookie`

5. Клієнт:

- оновлює `access_token` у памʼяті
- повторює початковий запит

## Logout

1. Клієнт викликає `/auth/logout`
2. Сервер:

- очищає cookie з `refresh_token`
- інвалідовує сесію в базі даних
- додає ексіс токен jti до denylist (база заблокованих токенів) - до таблиці заблокованих токенів

revoked_tokens

- jti: string
- revokedAt: Date
