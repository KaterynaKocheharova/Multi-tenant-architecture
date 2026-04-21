# ДОДАТКОВІ ГЛОБАЛЬНІ/СИСТЕМНІ РИЗИКИ

## 1. Ризик noisy neighbor у shared database

Один tenant з високим навантаженням може погіршити продуктивність для всіх інших tenant-ів.

**Запропоноване рішення:**

- Впровадити поділ SLO продуктивності як частину платформної архітектурної політики.

## 2. Ризик гранулярності відновлення на рівні tenant

Чисте відновлення даних одного tenant-а складніше, коли всі tenant-и працюють в одній спільній базі даних.

**Evidence:**

- shared database decision in docs/adr/ADR-002-data-isolation.md#L20
- backup policy in docs/nfr.md#L60

**Запропоноване рішення:**

- Визначити tenant-scoped стратегію відновлення та критерії приймання для сценаріїв часткового restore у DR-плануванні.

## 3. Ризик витоку міжtenant-метаданих через GLOBAL events

Навіть за коректної row isolation, глобально видимі записи можуть розкривати чутливі патерни організаційної активності.

**Evidence:**

- global-row visibility in docs/rls-details.md#L81
- global-row visibility in docs/rls-details.md#L91
- cross-tenant collaboration requirement in docs/adr/ADR-002-data-isolation.md#L39

**Запропоноване рішення:**

- Визначити модель класифікації global-data з явним переліком: які поля глобально видимі, а які обмежені.

## 4. Ризик складності політик у mixed-visibility RLS

Коректність стає важче доводити, коли SELECT та write-шляхи розходяться для GLOBAL і TENANT scope.

**Evidence:**

- mixed-table split policy design in docs/adr/ADR-002-data-isolation.md#L30
- participation/award write predicates in docs/rls-details.md#L122
- participation/award write predicates in docs/rls-details.md#L153

**Запропоноване рішення:**

- Розглядати набір RLS-політик як versioned security artifact з перевіркою сценаріїв як release gate.

## 5. Ризик розходження шарів авторизації при масштабуванні

RBAC/ABAC middleware та RLS можуть семантично дрейфувати з часом, що призводить до непослідовних allow/deny рішень між endpoint-ами.

**Evidence:**

- layered authorization model in docs/rbac.md#L106
- layered authorization model in docs/rbac.md#L114
- layered authorization model in docs/rbac.md#L142
- DB policy layer in docs/rls-details.md#L41

**Запропоноване рішення:**

- Вести єдиний каталог семантики авторизації, який зв'язує намір endpoint-а з API-перевірками та очікуваннями DB-політик.

## 6. Ризик залежності auth hot-path від глобальних перевірок

Перевірка revoked token на кожен запит може стати вузьким місцем за латентністю та доступністю при зростанні трафіку.

**Evidence:**

- denylist check for every request in diagrams/auth-sequence.mmd#L76
- docs/security.md#L32

**Запропоноване рішення:**

- Визначити performance budget для token validation та поведінку у failure-mode при деградації denylist-залежності.

## 7. Ризик blast radius при деплої

Відсутність deployment topology ускладнює розуміння, чи ізольовані відмови, чи вони платформного масштабу.

**Evidence:**

- deployment doc is empty in docs/deployment.md
- high-availability assumptions in docs/nfr.md#L18
- high-availability assumptions in docs/nfr.md#L67

**Запропоноване рішення:**

- Описати межі платформної топології (region, network, runtime, data plane) та failure domains у deployment-документації.

## 8. Ризик compliance/data-governance для міжtenant-потоків студентських даних

Платформа підтримує міжшкільну взаємодію, але governance-межі видимості персональних даних визначені неявно.

**Evidence:**

- cross-tenant collaboration in docs/README.md#L14
- student/participation entities in docs/data-model.md#L63
- student/participation entities in docs/data-model.md#L169
- audit logging requirements in docs/security.md#L68

**Запропоноване рішення:**

- Визначити governance matrix для обробки персональних даних за роллю, tenant boundary та event scope.
