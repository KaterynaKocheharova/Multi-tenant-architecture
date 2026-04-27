## 🛡️ Тут low-level огляд RLS від PostgreSQL та приклади імплементації, щоб продемонструвати, що його дійсно можна застосовувати.

### Базовий приклад:

Створюємо політику для запиту:

```sql
ALTER TABLE table_name ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON table_name
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

Транзакція запиту повинна встановити контекст тенанта та ролі:

```sql
SET LOCAL app.current_tenant = 'tenant-uuid-here';
SET LOCAL app.current_global_role = 'Member'; -- or 'Sysadmin'
```

### Налаштування RLS з відмовою за замовчуванням (немає контексту тенанта, забули його додати = немає даних)

```sql
ALTER TABLE table_name ENABLE ROW LEVEL SECURITY;
ALTER TABLE table_name FORCE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS tenant_isolation ON table_name;

CREATE POLICY tenant_isolation ON table_name
FOR ALL
USING (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND tenant_id = current_setting('app.current_tenant', true)::uuid
)
WITH CHECK (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND tenant_id = current_setting('app.current_tenant', true)::uuid
);
```

### Доступ sysadmin

Sysadmin має читати все

```sql
CREATE POLICY tenant_or_sysadmin_select ON table_name
FOR SELECT
USING (
  (
    current_setting('app.current_tenant', true) IS NOT NULL
    AND tenant_id = current_setting('app.current_tenant', true)::uuid
  )
  OR current_setting('app.current_global_role', true) = 'Sysadmin'
);
```

### Таблиці зі змішаною видимістю

**Політика читання** — будь-хто може бачити глобальні рядки;

```sql
CREATE POLICY event_select ON event
FOR SELECT
USING (
  (
    current_setting('app.current_tenant', true) IS NOT NULL
    AND tenant_id = current_setting('app.current_tenant', true)::uuid
  )
  OR scope = 'GLOBAL'
  OR current_setting('app.current_global_role', true) = 'Sysadmin'
);
```

**Міжтенантний запис участі для глобальних подій** — організатор з будь-якого тенанта повинен мати змогу реєструвати або оновлювати учасника з іншого тенанта, якщо подія має статус `GLOBAL`:

```sql
CREATE POLICY participation_select ON event_participation
FOR SELECT
USING (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND (
    participant_tenant_id = current_setting('app.current_tenant', true)::uuid
    OR EXISTS (
      SELECT 1 FROM event e WHERE e.id = event_id AND e.scope = 'GLOBAL'
    )
  )
);

CREATE POLICY participation_write ON event_participation
FOR INSERT, UPDATE
WITH CHECK (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND (
    -- tenant-scoped event: participant must belong to the same tenant
    participant_tenant_id = current_setting('app.current_tenant', true)::uuid
    OR
    -- global event: any authenticated tenant may write participation
    EXISTS (
      SELECT 1 FROM event e WHERE e.id = event_id AND e.scope = 'GLOBAL'
    )
  )
);
```

### Функція-обгортка для встановлення контексту БД

```typescript
async function withTenantContext<T>(
  prisma: PrismaClient,
  schoolId: string,
  fn: (tx: Prisma.TransactionClient) => Promise<T>,
): Promise<T> {
  return prisma.$transaction(async (tx) => {
    await tx.$executeRaw`SET LOCAL app.current_tenant = ${schoolId}`;
    return fn(tx);
  });
}
```
