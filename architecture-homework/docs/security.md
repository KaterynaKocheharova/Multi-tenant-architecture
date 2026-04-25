<!-- DOCS_NAV_START -->

[Docs Home](README.md) | [API Design](api-design.md) | [Auth](auth.md) | [RBAC](rbac.md) | [Data Model](data-model.md) | [Deployment](deployment.md) | [Containers](containers.md) | [Context](context.md) | [Frontend](front.md) | [NFR](nfr.md) | [Req-Res Propagation](req-res-propagation.md) | [Risks](risks.md)

<!-- DOCS_NAV_END -->

## Навігація в документі

- [SECURITY](#security)
  - [1. Magic Link двофакторна аутентифікація](#1-magic-link-двофакторна-аутентифікація)
  - [2. Хешування паролів](#2-хешування-паролів)
  - [3. Access Token - короткодіючий](#3-access-token---короткодіючий)
  - [4. Refresh Token - довгоживучий](#4-refresh-token---довгоживучий)
  - [5. Token Revocation (Denylist)](#5-token-revocation-denylist)
  - [6. Session Invalidation](#6-session-invalidation)
  - [7. Rate Limiting від DDoS та brute force](#7-rate-limiting-від-ddos-та-brute-force)
  - [8. HTTPS (TLS/SSL) ВІД Man-in-the-Middle](#8-https-tlsssl-від-man-in-the-middle)
  - [9. Tenant-based Access Control + RBAC + ABAC](#9-tenant-based-access-control--rbac--abac)
  - [10. Audit Logging](#10-audit-logging)
  - [11. Блокування айпі адрес при великій спробі невдалих запитів на автенцифікаційні ендпоїнти](#11-блокування-айпі-адрес-при-великій-спробі-невдалих-запитів-на-автенцифікаційні-ендпоїнти)
  - [12. SSL/TLS Certificate Validity - Автоматичне renewal перед expiry](#12-ssltls-certificate-validity---автоматичне-renewal-перед-expiry)
  - [13. Sanitization інпутів від користувача](#13-sanitization-інпутів-від-користувача)
  - [14. Vercel Security Headers Config](#14-vercel-security-headers-config)

<!-- DOCS_TOC_START -->
<!-- DOCS_TOC_END -->

## SECURITY

#### 1. Magic Link двофакторна аутентифікація

- **Мета**: Забезпечити надійний, user-friendly доступ без компрометованих паролів.
- **Від чого**: Якщо пароль вкрадуть або взгадають, зловмисникам його буде недостатньо.

#### 2. Хешування паролів

- Brute force захист. Сіль (salt) змушує хакера атакувати кожного користувача окремо, що робить масовий злам бази неможливим.
- Bcrypt навмисно робить хешування повільним, роблячи кожну спробу перебору часозатратною.

#### 3. Access Token - короткодіючий

- У зловмисників мало часу на використання токену, плюс у них нема рефреш токену на отримання нової пари.
- Додавання токену у denyList після вилогіну унеможливлює його перевикористання.

#### 4. Refresh Token - довгоживучий

- `HttpOnly` - JavaScript НЕ може отримати доступ - XSS захист
- `Secure` - передається тільки через HTTPS, не можна перехопити
- `SameSite=Lax` - знижує CSRF ризик і зменшує environment-specific проблеми інтеграції
- `Path=/auth` - додається лише до auth endpoint-ів
- `Domain=yourdomain.com` - указуємо з якого домену можна передавати кукі, щоб не відправляти при запитах із іншого сайту

#### 5. Token Revocation (Denylist)

- Якщо хтось захопив refresh token, вимушений logout його інвалідує.
- При кожному запиті перевіряється бд за токен jti, і якщо він "revoked", це унеможливлює запит

#### 6. Session Invalidation

- На користувача максимум **1-2 активні сесії**
- При логіні з нового пристрою, автоматично logout старого

#### 7. Rate Limiting від DDoS та brute force

- Кількість максимальних спроб на запити на логін, верифікацію лінки та рефреш
- Response при превищенні: HTTP 429 (Too Many Requests) + Retry-After header

#### 8. HTTPS (TLS/SSL) ВІД Man-in-the-Middle

- Шифрування всіх даних при передачі

#### 9. Tenant-based Access Control + RBAC + ABAC

- RLS
- Мідлвари для RBAC, ABAC

#### 10. Audit Logging

<a id="10-audit-logging"></a>

- Login/logout (успіх + failure)
- Доступ до sensitive data (reports, student grades)
- Rate limit violations
- Невдалі спроби доступу до чужих ресурсів

#### 11. Блокування айпі адрес при великій спробі невдалих запитів на автенцифікаційні ендпоїнти

#### 12. SSL/TLS Certificate Validity - Автоматичне renewal перед expiry

#### 13. Sanitization інпутів від користувача

Прибирати теги та атрибути коду при перевірці даних із інпуту, щоб не дозволити зловмисному скірпту виконатися.

#### 14. Vercel Security Headers Config

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "Content-Security-Policy",
          /* CSP — Головний щит. Визначає, звідки браузеру дозволено завантажувати ресурси.
             - default-src 'self': дозволяти ресурси тільки з власного домену.
             - script-src: дозволяє скрипти з власного сайту, інлайнові скрипти ('unsafe-inline') та конкретний довірений домен.
             - style-src: дозволяє стилі сайту та інлайнові стилі (часто потрібно для React).
             - img-src: дозволяє зображення з власного сайту та base64 дані (data:).
             - connect-src: КРИТИЧНО — тут має бути адреса API, щоб запити не блокувалися. */
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' https://trusted-scripts.com; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' https://your-nestjs-api.com;"
        },
        {
          "key": "X-Frame-Options",
          /* Забороняє відображати сайт у <iframe> на сторонніх ресурсах.
             Захищає від Clickjacking. */
          "value": "SAMEORIGIN"
        },
        {
          "key": "X-Content-Type-Options",
          /* Забороняє браузеру виконувати файл, якщо його MIME-тип відрізняється від заявленого. */
          "value": "nosniff"
        },
        {
          "key": "Referrer-Policy",
          /* 'strict-origin-when-cross-origin' — передає повну адресу всередині сайту,
             але тільки домен при переході на зовнішні ресурси по HTTPS. */
          "value": "strict-origin-when-cross-origin"
        },
        {
          "key": "Strict-Transport-Security",
          /* HSTS — примушує браузер використовувати тільки HTTPS протягом наступного року.
             includeSubDomains — поширюється і на піддомени.
             preload — дозвіл на включення сайту до preload-списку браузерів. */
          "value": "max-age=31536000; includeSubDomains; preload"
        },
        {
          "key": "Permissions-Policy",
          /* Обмежує доступ до апаратних функцій пристрою. */
          "value": "camera=(), microphone=(), geolocation=()"
        }
      ]
    }
  ]
}
```
