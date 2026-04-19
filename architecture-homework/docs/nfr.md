# Non-Functional Requirements (NFR)

## 📊 PERFORMANCE

### Database Query Optimization

- **Indexes**: На `schoolId`, `userId`, `eventId`, `createdAt` (composite indexes для common queries)
- **N+1 Query Prevention**: Eager loading / batch loading (DataLoader для GraphQL)
- **Caching**:
  - Redis для user profiles (TTL 1 час)
  - Redis для report list (TTL 5 хвилин)
  - Cache invalidation при змінах

### Frontend Performance

- **Bundle size**: < 300 KB gzipped
- **First Contentful Paint**: < 2 сек
- **Time to Interactive**: < 4 сек

---

## 🚀 AVAILABILITY & RELIABILITY

### Uptime SLA

- **Target**: 99.5% (4 часа downtime на місяць допускається)
- **Рекомендація**: 99.9% якщо можливо (43 хвилини на місяць)

### Error Handling

- **Graceful degradation**: Якщо Redis недоступний, система повинна працювати (без caching)
- **Circuit Breaker pattern**: Для зовнішніх API (email, file storage)
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
- **Recovery Time Objective (RTO)**: < 1 часа
- **Recovery Point Objective (RPO)**: < 15 хвилин
- **테스ト**: Щомісяця робити dry-run restore

### Health Checks & Monitoring

- **Liveness probe** (`GET /health`): Система запущена
- **Readiness probe** (`GET/ready`): БД + Redis доступні
- **Metrics**: CPU, memory, disk, request rate, error rate
- **Alerts**: На 90% disk usage, CPU > 80% for 5 хвилин, > 1% 5xx errors

---

## 📈 SCALABILITY

### Horizontal Scaling

- **Stateless API servers**: Добавлити нові instances без перезагрузки
- **Load balancing**: Round-robin або least-connections
- **Database connection pooling**: Pgbouncer або вбудована (max 20 connections per server)

### Database Sharding (для майбутнього)

- **Шардування по tenantId**: Кожна школа -> однієї шарді
- **Коли потребується**: > 1M записів або > 1000 тенантів
- **Перехідна крок**: Single database з гарною оптимізацією (індекси, partitioning)
- **PostgreSQL Partitioning**: Партиціонування tables по schoolId (автоматична оптимізація)

### Hybrid Model (Multi-Tenant + Single-Tenant)

- **Multi-tenant** (Shared): Малі школи, < 100 користувачів
- **Single-tenant** (Dedicated): Великі школи, > 500 користувачів або спеціальні вимоги
- **Переведення**: Механізм міграції даних з shared на dedicated instance

---

## 🛠️ RELIABILITY

### Error Recovery

- **Автоматичне переспроба** failed jobs (email, report generation)
- **Idempotency**: POST /users при реселенді повинна повернути 200 (не 400)
- **Queue система** (Bull, RabbitMQ) для async jobs:
  - Send emails
  - Generate reports
  - File processing

### Data Consistency

- **ACID compliance**: PostgreSQL гарантує
- **Eventual consistency** для read replicas (< 1 сек lag допускається)
- **Race condition prevention**: Optimistic locking на critical updates
  ```sql
  UPDATE reports SET status = 'completed', version = version + 1
  WHERE id = :reportId AND version = :expectedVersion
  ```

### Testing

- **Unit tests**: > 80% code coverage
- **Integration tests**: Всі API endpoints + auth flows
- **Load testing**: min 1000 concurrent users
- **Chaos engineering**: Периодично тестувати відключення БД, Redis, terchoвиx сервісів

---

## 🌍 COMPLIANCE & DATA PROTECTION

### GDPR Compliance (якщо користувачі з ЄС)

- **Right to be forgotten**: Вибір даних користувача + удалити з резервних копій за 30 днів
- **Data Export**: Користувач може експортувати свої дані (JSON)
- **Data Processing Agreement (DPA)**: З усіма sub-processors

### Compliance for Ukraine

- **PERSONAL DATA PROTECTION**: Закон України про захист персональних даних
- **Retention policy**: Обов'язково вказати у Privacy Policy
- **Шифрування**: At-rest encryption для sensitive data (passwords, email, studentDetails)

### Encryption at Rest

- **Database**: Ціле encryption (AWS RDS encryption) або на disk (dm-crypt)
- **Backup**: Обов'язково зашифрований
- **Algorithm**: AES-256

---

## 👥 USABILITY & ACCESSIBILITY

### UI/UX Requirements

- **Response time**: UI повинен реагувати за < 100ms (immediat feedback)
- **Mobile-friendly**: Responsive design (мобільні вчителі можуть вводити дані)
- **Accessibility**: WCAG 2.1 Level AA мінімум
  - Color contrast ratio > 4.5:1
  - Keyboard navigation
  - Screen reader support

### Localization

- **Ukrainian support**: Основна мова
- **Future**: Англійська, Російська (якщо потребується)

---

## 📋 MAINTAINABILITY & DOCUMENTATION

### Code Quality

- **Linting**: ESLint + Prettier (auto-format)
- **Type safety**: TypeScript strict mode
- **Documentation**: JSDoc + README для кожного module
- **Logging**: Структуровані logs (JSON format) для easy parsing

### Version Control & Deployment

- **Semantic Versioning**: MAJOR.MINOR.PATCH
- **CI/CD Pipeline**:
  - Автоматичні тести при push
  - Build + deploy на staging
  - Manual approval для production
- **Rollback capability**: < 5 хвилин rollback до попередної версії

---

## 🔌 INTEROPERABILITY & API DESIGN

### API Versioning

- **Strategy**: URL-based versioning (`/api/v1`, `/api/v2`)
- **Deprecation**: Підтримувати старі версії мінімум 6 місяців
- **Backwards Compatibility**: При додаванні полів - додавайте нові, не видаляйте старі
- **Breaking Changes**: Тільки у мажорних версіях, з 6-місячною deprecation notice

### Data Format Standards

- **Request/Response**: JSON (не XML, SOAP тощо)
- **Date Format**: ISO 8601 (UTC timezone)
- **Pagination**:
  ```json
  {
    "data": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "hasMore": true
    }
  }
  ```
- **Error Format**:
  ```json
  {
    "error": {
      "code": "INVALID_REQUEST",
      "message": "User already exists",
      "details": {...}
    }
  }
  ```

### Third-party Integration

- **Email Service**: Mailgun, SendGrid (для magic links, notifications)
- **File Storage**: AWS S3, Google Cloud Storage (для video uploads, certificates)
- **Logging Service**: ELK Stack, Datadog (для centralized logging)
- **Payment** (якщо потребується): Stripe API

---

## ⚙️ RESOURCE MANAGEMENT & QUOTAS

### Per-Tenant Resource Limits

- **Users**:
  - Малі школи: max 100 користувачів
  - Середні: max 500
  - Великі: unlimited (single-tenant)
- **Events**: max 1000 per month
- **Reports**: max 100 в черзі одночасно
- **Storage**:
  - Video uploads: max 10 GB per tenant
  - Documents: max 5 GB
- **API Calls**: max 1000 requests per hour per tenant (за пік)

### Rate Limiting Per Tenant

- **Tier-based**:
  - Free: 100 API calls/hour
  - Pro: 1000 API calls/hour
  - Enterprise: unlimited
- **Burst limit**: Дозволити 150% протягом 5 хвилин (для спайків)

### Database Connection Pooling

- **Max connections**: 20 per server
- **Idle timeout**: 5 хвилин
- **Queue**: Якщо no free connections, чекати max 5 сек, потім reject з HTTP 503

### Memory & CPU Constraints

- **API container**: max 512 MB RAM
- **Worker container** (report generation): max 1 GB RAM
- **CPU limit**: Завізати на infra spec, але мати capability auto-scale

---

## 📊 BATCH PROCESSING & ASYNC OPERATIONS

### Report Generation

- **Queue-based processing**: Bull, RabbitMQ, або AWS SQS
- **TTL**: Job має завершитися за < 30 сек (для UI), або > 30 сек -> background job
- **Retry logic**:
  - Failed job retry 3 times з exponential backoff
  - Після 3 fails -> move to dead letter queue + alert admin
- **Progress tracking**: WebSocket updates або polling (`GET /reports/:id/status`)

### Bulk Operations

- **Batch import**: CSV upload для students, events
  - Max file size: 5 MB (до ~1000 rows)
  - Validation: Перед import перевірити всі rows (не імпортити partial)
  - Error reporting: Вказати точно які rows failed + reason
- **Batch export**: CSV download для reports

### Email Queue

- **Queue**: Усі emails мають йти через queue (НЕ synchronous!)
- **Retry**: Переспробити при failure (exponential backoff)
- **Tracking**: Логувати status (sent, failed, bounced)

### Scheduled Tasks

- **Cron jobs** (за допомогою node-cron або AWS Lambda):
  - Генерування reminder emails (напр., за день до deadline)
  - Очищення старих revoked_tokens з БД
  - Синхронізація з external APIs
  - Генерування analytics
- **Monitoring**: Логувати result кожного job execution

---

## 🔍 SEARCH & QUERY PERFORMANCE

### Full-Text Search

- **For**: Students, teachers, events, lesson plans
- **Index**: PostgreSQL GIN indexes для full-text search
- **Response time**: < 500ms для 1000+ результатів
- **Relevance**: Better match scores повинні бути first

### Filtering & Sorting

- **Performance**: На 100k+ records мають бути < 100ms
- **Indexed fields**: `schoolId`, `createdAt`, `status`, `userId`
- **Compound queries**: (schoolId + status + userId) повинні бути indexed

### Pagination

- **Cursor-based** (для production scalability) замість offset-based
  ```
  GET /events?cursor=abc123&limit=20
  ```

---

## 🔄 DATA MIGRATION & EVOLUTION

### Schema Evolution

- **Zero-downtime migrations**: Додавайте nullable columns перед їх використанням
- **Deprecation**: Помічайте стовпці як deprecated на UI, потім видаліть в наступній версії
- **Testing**: Кожна міграція повинна мати up/down функціональність

### Tenant Data Migration

- **Multi-tenant to Single-tenant**: Механізм експорту тенанта в окремий інстанс
- **Data export**: Користувач повинен мати можливість експортувати свої дані (GDPR requirement)
- **Backup & restore**: Механізм для restore якщо щось пішло не так

### Version Upgrade Path

- **Compatibility matrix**: Які backend versions совместимі з якими frontend versions
- **Auto-upgrade**: Frontend повинен мати option auto-update version
- **Graceful downgrade**: Якщо frontend newer ніж backend, мати fallback behavior

---

## 🔔 REAL-TIME CAPABILITIES

### Notifications

- **Push notifications**: Для новых результатів конкурсу, успішного upload video
- **Email notifications**: За важливі events (new attendance, grades assigned)
- **In-app notifications**: Real-time toast/banner у UI
- **Frequency**: Max 1 notification per 5 хвилин per user (уникнути spam)

### Live Updates

- **WebSocket connection**: Для live event participant list, real-time jury scoring
- **Reconnection logic**: Auto-reconnect при disconnect (з exponential backoff)
- **Message queuing**: Якщо client offline, queue messages до reconnection
- **Graceful fallback**: Якщо WebSocket not supported, polling (`GET /events/:id/updates`)

### Real-time Analytics

- **Dashboard metrics**:
  - Active users per school (real-time)
  - Events happening now
  - Reports being generated
- **Latency**: Updates < 5 сек

---

## 🌐 ENVIRONMENT CONFIGURATION & PORTABILITY

### Environment Management

- **Configurations**:
  - `development`: Verbose logging, no rate limits, localhost
  - `staging`: Production-like, aber with test data
  - `production`: High security, monitoring enabled
- **Config file**: `.env` (не committuyте credentials!)
- **Secrets management**: Use AWS Secrets Manager, HashiCorp Vault, або Kubernetes Secrets

### Database Migrations

- **Portability**: Код повинен бути database-agnostic (легко мігрувати з PostgreSQL на інший DB)
- **Seed data**: Для кожного environment інші seed scripts
- **Connection pooling**: Налаштовуватися per-environment

### Docker & Container Orchestration

- **Docker image**: Repeatable builds, pinned versions
- **Kubernetes (optional)**: Для auto-scaling + self-healing
- **Resource requests**: CPU 100m, Memory 256Mi (for development)

---

## 💰 COST EFFICIENCY

### Multi-Tenancy Benefits

- **Shared infrastructure**: Один database instance for many schools (vs dedicated per school)
- **Resource pooling**: CPU/memory розподіляється ефективно
- **Monitoring**: Per-tenant resource usage tracking (для future billing)

### Optimization

- **Database indexes**: Правильні indexes = менше CPU for queries
- **Caching**: Redis cache vs re-computation = менше DB load
- **CDN**: Статичні assets (CSS, JS, images) з CDN (CloudFront, Cloudflare)
- **Reserved instances**: AWS Reserved Instances на 1-3 року for stable workloads

---

## 📈 OBSERVABILITY & MONITORING

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

### Distributed Tracing

- **Correlation ID**: На кожен request, pass through усі microservices
- **Tool**: Jaeger або AWS X-Ray
- **Purpose**: Зрозуміти full request path при проблемах

---

## 🧪 TESTING & QUALITY ASSURANCE

### Test Coverage

- **Unit tests**: > 80% для critical path (auth, validation, business logic)
- **Integration tests**: Всі API endpoints + database interaction
- **End-to-end tests**: Key user flows (login -> create event -> generate report)
- **Performance tests**: Ensure endpoints meet < 200ms requirement
- **Security tests**: SQL injection, XSS, CSRF protection

### Load Testing

- **Tool**: k6, Apache JMeter, або Locust
- **Scenario**: Симулюйте 100 concurrent users, каждый робить:
  - Login + fetch events + create report
  - Duration: 5 хвилин
- **Success criteria**: < 5% error rate, p95 < 500ms

### Browser Compatibility

- **Supported**: Chrome (last 2 versions), Firefox, Safari, Edge
- **Mobile**: iOS Safari, Chrome Mobile
- **Fallback**: Graceful degradation for older browsers (IE не потрібен)

---

## 📦 COMPATIBILITY & STANDARDS

### Protocol Standards

- **HTTP/2**: Мінімум
- **REST API**: HAL или JSON:API format (optional, але рекомендуємо consistency)
- **CORS**: Explicitly whitelist frontend domains (НЕ '\*')

### Browser APIs Used

- **localStorage**: Для preferences (не для sensitive data!)
- **sessionStorage**: Для temporary UI state
- **IndexedDB**: Якщо offline support потребується
- **Service Workers**: Для offline capabilities + push notifications

### Accessibility Standards

- **WCAG 2.1 Level AA**: Мінімум
- **Keyboard navigation**: Всі interactive elements повинні бути accessible via Tab
- **Screen reader**: ARIA labels для images, buttons
- **Color contrast**: 4.5:1 для text, 3:1 для UI components

---

## 🛡️ SECURITY CHECKLIST (OWASP Top 10)

1. ✅ **Injection** - Parameterized queries, input validation
2. ✅ **Broken Authentication** - Magic link + JWT + refresh token
3. ✅ **Sensitive Data Exposure** - HTTPS, encryption at rest, no logging of passwords
4. ✅ **XML External Entities** - Не використовуєте XML
5. ✅ **Broken Access Control** - RBAC, tenant isolation, audit logging
6. ✅ **Security Misconfiguration** - Environment-specific config, security headers
7. ✅ **Cross-Site Scripting (XSS)** - CSP, token in-memory, output encoding
8. ✅ **Insecure Deserialization** - JSON only, input validation
9. ✅ **Using Components with Known Vulnerabilities** - Dependency scanning (npm audit)
10. ✅ **Insufficient Logging & Monitoring** - Audit trail, alerts, centralized logging

---

## 🏆 SECURITY COMPLIANCE & CERTIFICATION

### Industry Standards Compliance

- **ISO/IEC 27001** (якщо потребується):
  - Information Security Management System
  - Документування всіх security processes
  - Регулярні security audits (2x в рік мінімум)
- **SOC 2 Type II** (для enterprise customers):
  - Security, availability, processing integrity
  - 6-місячна audit період

### Data Protection Laws Compliance

- **GDPR** (EU users):
  - Privacy Policy + Cookie Consent
  - Data Subject Rights (access, deletion, portability)
  - Data Processing Agreement з усіма vendors
  - Notification of breaches within 72 hours
- **Ukraine Data Protection Law**:
  - Приватність користувачів
  - Retention policy для персональних даних
  - Right to be forgotten

### Dependency Security

- **Supply Chain Security**:
  - `npm audit` регулярно (бути < 10 vulnerabilities)
  - Pinned versions для critical dependencies (не floating versions)
  - Automated updates через Dependabot
  - Security patches applied within 7 days (high/critical) або 30 days (medium)

### Security Assessment Checklist

- **Quarterly Security Review**:
  - Перевірити всі NFR вимоги (особливо Security)
  - Перегляд audit logs на suspicious activity
  - Penetration testing (1x на рік мінімум)
  - Vulnerability scan database (1x на місяць)
- **Incident Response Plan**:
  - Response team + escalation path
  - Time to notify users if breach detected
  - Post-incident review процес

### Security Training & Awareness

- **Developer Training**:
  - OWASP Top 10 знання
  - Secure coding practices
  - Annually updated training
- **User Awareness**:
  - Security tips в app (як розпізнати phishing)
  - Password best practices
  - Two-factor authentication benefits

### Red Flags & Monitoring

- **Suspicious Activity Alerts**:
  - 10+ failed login attempts in 1 hour -> lock account
  - Multiple concurrent sessions from different geographies -> ask verification
  - Mass data export -> alert admin
  - Role change by non-admin -> alert user
  - Deleted records -> audit trail + admin notification

---

## 📊 SECURITY AS NFR: SUMMARY

| Аспект                | Вимога                                     | Відповідальність   |
| --------------------- | ------------------------------------------ | ------------------ |
| **Encryption**        | All data in transit (HTTPS) + at rest (DB) | DevOps + Backend   |
| **Authentication**    | Multi-factor (magic link), secure tokens   | Backend + Security |
| **Authorization**     | RBAC + tenant isolation + audit trail      | Backend + QA       |
| **Monitoring**        | Real-time alerts + audit logging           | DevOps + Backend   |
| **Incident Response** | < 1 hour detection, < 72h notification     | All team           |
| **Compliance**        | GDPR + Ukraine law + internal policies     | Legal + Backend    |
| **Testing**           | Security tests + penetration testing       | QA + Security      |
| **Documentation**     | Security policy + runbooks                 | All team           |

**Висновок**: Security це не додатковий feature, це **foundation** всієї системи. Все інше (Performance, Scalability) не має значення якщо система скомпрометована.
