# Non-Functional Requirements (NFR)

## RESILIENCY

### Fault Tolerance

Система повинна працювати зі зменшеною функціональністю, а не повністю падати:

```
Сценарій: Redis (cache) недоступний
❌ Система падає з 500 error
✅ Система запитує дані з БД (повільніше, але працює)

Сценарій: Email service недоступний
❌ Бекенд crash при спробі відправити email
✅ Email додається в queue, повторюється пізніше або використовується бекапний сервіс

Сценарій: One API server упав
❌ Load balancer розуміє, що 50% servers down
✅ Load balancer відправляє запити на інші 3 servers
```

### Circuit Breaker Pattern

Для запобігання "crashing cascade" при failure зовнішніх сервісів:

```
Стани:
CLOSED (нормально): запити проходять
  → Якщо помилки → перехід в OPEN

OPEN (відключено): запити не проходять, повернути error одразу
  → Після timeout → перейти в HALF_OPEN

HALF_OPEN (тестування): дозволити 1 запит
  → Якщо успіх → повернути в CLOSED
  → Якщо fail → повернути в OPEN

Приклад: Email service timeout
- Перша помилка: CLOSED
- 5 помилок за 30 сек: перейти в OPEN
- Не намагатися відправити email наступні 5 хвилин
- Після 5 хвилин: HALF_OPEN, спробувати знову
- Якщо email успішно відправлений: повернути в CLOSED
```

### Error Handling

- **Retry logic** з exponential backoff:
  ```
  Спроба 1: immediately
  Спроба 2: 1 сек
  Спроба 3: 2 сек
  Спроба 4: 4 сек
  Спроба 5: 8 сек
  ```

### Database Backup & Disaster Recovery

- **Backup frequency**: Щодня (auto-backup)
- **Retention**: Мінімум 30 днів

## SCALABILITY

### Horizontal Scaling

- **Stateless API servers**: Добавлити нові instances без перезагрузки
- **Load balancing**: щоб централізовано перенапралвяти запити на найбільш підходящий сервер

### Database

- **Primary path (current)**: shared PostgreSQL + RLS + table partitioning by `tenantId`.
- **Conditional future path**: tenant sharding (school -> separate DB/server) is allowed only if triggers are met:
  - sustained p95 latency > 500ms after indexing/partition tuning,
  - repeated noisy-neighbor incidents affecting SLA,
  - one tenant exceeds agreed storage or throughput threshold.
- **Migration gate**: sharding is a planned future state, not current architecture.

## OBSERVABILITY

### Logging Levels

```
DEBUG: Detailed info (variable values, loop iterations) - DEV only
INFO: General flow (user login, API request started)
WARN: Unexpected but recovered (retry #2, low disk space)
ERROR: Unexpected situation (API request failed, auth denied)
FATAL: System cannot continue (database down, out of memory)
```

### Structured Logging Format

```json
{
  "timestamp": "2026-04-19T10:30:45Z",
  "level": "ERROR",
  "service": "auth-service",
  "userId": "uuid",
  "tenantId": "uuid",
  "message": "Failed to send magic link",
  "error": "Email service timeout",
  "duration_ms": 5000,
  "traceId": "xyz123"
}
```

### Metrics to Track

- **Business metrics**:
  - New registrations per day
  - Active users per school
  - Report generation success rate
  - Event participation rate
- **Technical metrics**:
  - API response time (p50, p95, p99)
  - Error rate per endpoint
  - Database query time
  - Queue job processing time
  - Cache hit rate

### Alerting

- **Critical**:
  - Database connection pool exhausted
  - 5xx error rate > 1%
  - Disk space < 10%
  - Memory usage > 90%
- **Warning**:
  - Response time p95 > 500ms
  - CPU > 80% for 5 хвилин
  - Queue backup > 1000 jobs
