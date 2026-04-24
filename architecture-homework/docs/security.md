<!-- DOCS_NAV_START -->
[Docs Home](README.md) | [API Design](api-design.md) | [Auth](auth.md) | [RBAC](rbac.md) | [Data Model](data-model.md) | [Security](security.md) | [Deployment](deployment.md) | [Containers](containers.md) | [Context](context.md) | [Frontend](front.md) | [NFR](nfr.md) | [Req-Res Propagation](req-res-propagation.md) | [Risks](risks.md)
<!-- DOCS_NAV_END -->

## Навігація в документі

- [🔒 SECURITY](#🔒-security)
  - [Authentication & Authorization](#authentication-authorization)
    - [1. Magic Link двофакторна аутентифікація](#1-magic-link-двофакторна-аутентифікація)
    - [2. Хешування паролів](#2-хешування-паролів)
    - [3. Access Token (JWT) - короткодіючий](#3-access-token-jwt-короткодіючий)
    - [4. Refresh Token - довгоживучий, як HttpOnly Cookie](#4-refresh-token-довгоживучий-як-httponly-cookie)
    - [5. Token Revocation (Denylist)](#5-token-revocation-denylist)
    - [6. Session Invalidation & Concurrent Session Limit](#6-session-invalidation-concurrent-session-limit)
    - [7. Rate Limiting від DDoS та brute force](#7-rate-limiting-від-ddos-та-brute-force)
    - [8. HTTPS (TLS/SSL) ВІД Man-in-the-Middle (MITM)](#8-https-tlsssl-від-man-in-the-middle-mitm)
    - [9. Tenant-based Access Control](#9-tenant-based-access-control)
    - [10. Row-Level Security (RLS) в БД](#10-row-level-security-rls-в-бд)
    - [12. Обов'язкові Security Headers](#12-обовязкові-security-headers)
    - [13. Audit Logging](#13-audit-logging)
    - [14. Блокування айпі адрес при великій спробі невдалих запитів на автенцифікаційні ендпоїнти.](#14-блокування-айпі-адрес-при-великій-спробі-невдалих-запитів-на-автенцифікаційні-ендпоїнти)
    - [15. SSL/TLS Certificate Validity - Автоматичне renewal перед expiry](#15-ssltls-certificate-validity-автоматичне-renewal-перед-expiry)
    - [16. Sanitization](#16-sanitization)

<!-- DOCS_TOC_START -->
<!-- DOCS_TOC_END -->


## 🔒 SECURITY

### Authentication & Authorization

#### 1. Magic Link двофакторна аутентифікація

- **Мета**: Забезпечити надійний, user-friendly доступ без компрометованих паролів.
- **Від чого**: Якщо пароль вкрадуть, зловмисникам його буде недостатньо.

#### 2. Хешування паролів

- Brute force (Брутфорс): Спроба вгадати пароль шляхом перебору всіх комбінацій. Сіль (salt) змушує хакера атакувати кожного користувача окремо, що робить масовий злам бази неможливим.
- Rainbow table (Райдужні таблиці): Використання гігантських словників із заздалегідь прорахованими хешам.Сіль робить ці таблиці марними, оскільки додає унікальний рандомний рядок до пароля, змінюючи фінальний хеш.
- GPU/ASIC cracking: Використання спеціалізованого потужного обладнання для прискореного перебору. Bcrypt вирішує це через навмисну повільність (cost factor), роблячи кожну спробу перебору дорогою за часом.

#### 3. Access Token (JWT) - короткодіючий

- У зловмисників мало часу на використання токену, плюч у них нема рефреш токену на отримання нової пари.
- Додавання токену у denyList після вилогіну унеможливлює його перевикористання.

#### 4. Refresh Token - довгоживучий, як HttpOnly Cookie

- `HttpOnly` - JavaScript НЕ може отримати доступ - XSS захист
- `Secure` - передається тільки через HTTPS, не можна перехопити
- `SameSite=Lax` - знижує CSRF ризик і зменшує environment-specific проблеми інтеграції
- `Path=/auth` - додається лише до auth endpoint-ів
- `Domain=yourdomain.com` - указуємо з якого домену можна передавати кукі, щоб не відправляти при запитах із іншого сайту

#### 5. Token Revocation (Denylist)

- Якщо хтось захопив refresh token, вимушений logout його інвалідує.
- При кожному запиті перевіряється бд за токен jti, і якщо він "revoked", це унеможливлює запит

#### 6. Session Invalidation & Concurrent Session Limit

- На користувача максимум **1-2 активні сесії** (НЕ рекомендуємо 10+)
- При логіні з нового пристрою, автоматично logout старого

#### 7. Rate Limiting від DDoS та brute force

- Кількість максимальних спроб на хапити на логін, верифікацію лінки та рефреш
- Response при превищенні: HTTP 429 (Too Many Requests) + Retry-After header

#### 8. HTTPS (TLS/SSL) ВІД Man-in-the-Middle (MITM)

- Шифрування всіх даних при передачі

#### 9. Tenant-based Access Control

- Мідлвари що перевіряє доступ до ендпоїнту за тенант аід користувача

#### 10. Row-Level Security (RLS) в БД

#### 12. Обов'язкові Security Headers

- `Content-Security-Policy` (CSP): Запобігнення XSS
  ```
  Content-Security-Policy:
    default-src 'self';
    script-src 'self' trusted-cdn.com;
    style-src 'self' 'unsafe-inline'
  ```
- `X-Content-Type-Options: nosniff` (запобігнення MIME type sniffing)
- `X-Frame-Options: DENY` або `SAMEORIGIN` (запобігнення clickjacking)
- `X-XSS-Protection: 1; mode=block` (legacy, але корисне)
- `Referrer-Policy: strict-origin-when-cross-origin` (запобігнення витоку реферера)

#### 13. Audit Logging

- Login/logout (успіх + failure)
- Доступ до sensitive data (reports, student grades)
- Rate limit violations
- Невдалі спроби доступу до чужих ресурсів

#### 14. Блокування айпі адрес при великій спробі невдалих запитів на автенцифікаційні ендпоїнти.

#### 15. SSL/TLS Certificate Validity - Автоматичне renewal перед expiry

#### 16. Sanitization

Прибирати теги та атрибути коду при перевірці даних із інпуту, щоб не дозволити зловмисному скірпту виконатися.
