<!-- DOCS_NAV_START -->

[Docs Home](README.md) | [API Design](api-design.md) | [Auth](auth.md) | [RBAC](rbac.md) | [Data Model](data-model.md) | [Security](security.md) | [Deployment](deployment.md) | [Containers](containers.md) | [Context](context.md) | [Frontend](front.md) | [NFR](nfr.md) | [Req-Res Propagation](req-res-propagation.md) | [Risks](risks.md)

<!-- DOCS_NAV_END -->

## Навігація в документі

- [Tenant Resolution + Tenant-Scoped API Calls](#tenant-flow)
- [Post-Login Redirect by Role](#post-login-redirect-by-role)
- [Route Protection](#route-protection)
- [Navigation Menu](#navigation-menu)
  - [SYSADMIN](#sysadmin)
  - [SCHOOL_ADMIN](#school_admin)
  - [TEACHER](#teacher)
- [Access Token Lifecycle](#access-token-lifecycle)
- [State management](#state-management)
- [Api integration pattern](#api-integration-pattern)
  - [Що робить request interceptor](#що-робить-request-interceptor)
  - [Що робить response interceptor](#що-робить-response-interceptor)
  - [Flow обробки відповіді](#flow-обробки-відповіді)

<!-- DOCS_TOC_START -->
<!-- DOCS_TOC_END -->

## Tenant flow

1. Отримали юзер на фронті.
2. При кожному запиту на бек, передаємо токен.
3. Без отримує айді юзера з пейлоуду, шукає юзера.
4. Маючи тенант айді юзера, він тепер контролює запити до бази даних і додає відповідну фільтрацію.

Тобто по суті тенант запити і дані, що має отримати юзер, контролюються беком. А фронтенд контролює доступи до певних сторінок на основі ролей та автентифікації.

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
2. Чи юзер має ролі, що допускаються для поточного рауту?  
   → Якщо ні, то redirect на `/home`

## 403 Protection

Якщо юзер спробує перейти на сторінку конкретного ресурсу що йому не належить? Учитель відкриє `students/5`, хоча студент 5 йому не належить?

Додати інтерсептор, що перехопить 403 помилку. Перекидати або на хоум, або на логін, або показувати нот фаунд сторінку.

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
- My Profile (→ `/profile/me`)

### TEACHER

- Events (→ `/events`)
- Students (→ `/students`)
- Lesson Plans (→ `/lesson-plans`)
- Reports (→ `/reports`)
- My Profile (→ `/profile/me` / `/teachers/:userId`)

## Access Token Lifecycle

1. При переході лінкою у імейлі запит на верифікацію до бека, який дає токен.
2. Токен зберігаємо in-memory, не в локал сторідж щоб не можна було вкрасти.
3. При кожному запиті токен додається і валідуєтья на беці.
4. Якщо помилка 401 від бека у interceptor зробити рефреш `POST /auth/refresh`.
5. Якщо рефреш фейлиться, редіректити до сторінки логіну `/login`.
6. Якщо юзер вилогінюється, треба почистити токен після запиту логіну і редірект на `/login`.

## State management

Використовуватимемо Zustand, адже він

- швидко імплементовується
- забезпечує зручність маніпуляції глобальними даними

Зберігатиму там:

- інфу по юзеру
- інфу по поточному змаганню, де людина бере участь або організовує наразі
- токени сесії in-memory (`accessToken`)
- системна інфа типу: `isAuthenticated`
- інше

## Api integration pattern

Використовуватимемо React Query для керування асинхронними станами, кешування і background refetch.

Практично це дає нам:

- автоматичні стани `isLoading` / `isError` / `isSuccess` без ручного менеджменту в компонентах
- кешування відповідей що не рефетчити дані при кожному маунті анмаунті та при переході зі сторінки на сторінку
- повторний фетч у фоні при поверненні на вкладку або інвалідації кешу
- централізоване оновлення даних після мутацій через `invalidateQueries`
- контроль `staleTime` / `cacheTime` для балансу між актуальністю і кількістю запитів

Axios використовуватимемо як HTTP-клієнт із централізованими interceptors.

### Що робить request interceptor

- додає Authorization header з accessToken для всіх захищених запитів

### Що робить response interceptor

- перехоплює помилку
- один раз запускає refresh через POST /auth/refresh
- якщо refresh успішний: оновлює accessToken in-memory і повторює початковий запит
- якщо refresh неуспішний: очищає auth state і робить redirect на /login
- для 403, 404, 500 повертає нормалізовану помилку для UI

### Flow обробки відповіді

1. Компонент викликає API через React Query.
2. Axios request interceptor додає токен; tenant context для авторизації визначається бекендом.
3. Бекенд повертає відповідь.
4. Якщо відповідь 2xx: дані потрапляють у кеш React Query.
5. Якщо відповідь 401: response interceptor викликає refresh.
6. Якщо refresh успішний: початковий запит автоматично повторюється.
7. Якщо refresh неуспішний: logout, очищення in-memory токена, redirect на /login.
