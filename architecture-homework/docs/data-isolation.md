# Data Isolation Strategy

## Overview

This system uses a **shared database multi-tenancy model** with strict isolation enforced at the database level.

Each tenant (school) is logically isolated using a `tenantId` (`schoolId`) while still enabling **controlled cross-tenant collaboration** (e.g., webinars, competitions).

## 🏗️ Architecture Choice

### Shared Database

- Single PostgreSQL database
- Shared tables across all tenants
- Logical isolation via `tenantId`
- Database-level enforcement using Row-Level Security (RLS)
- Application-level safeguards for additional protection

## Why

### 1. Product Requirements Fit

The system requires some **cross-tenant features**: competitions and webinars

A shared database enables simple queries and avoids distributed system complexity.
If we used multiple databases and need to fetch a specific event participants, we would need to list all dbs, conduct fetching queries for each of them, and finally merge results in application level. In out shared db case we fetch all data with a simple request.

### 2. Fast Development

- single schema
- one migration pipeline

### 3. Operational Simplicity

- centralized monitoring
- simpler backups
- easier maintenance
- quick tenant onboarding

### 4. Cost Efficiency

- one database instance
- efficient resource usage
- no idle infrastructure per tenant (if we used multiple dbs, we would need to pay for each even if data is not used and activity is minimal)

## 🛡️ Isolation Mechanisms

### 1. Row-Level Security (Primary Enforcement)

All tenant-scoped tables must enforce RLS.

Basic example:

```sql
ALTER TABLE table_name ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON table_name
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

Each request transaction must set tenant and role context:

```sql
SET LOCAL app.current_tenant = 'tenant-uuid-here';
SET LOCAL app.current_global_role = 'Member'; -- or 'Sysadmin'
```

### When to set

Recommended order:

1. resolve tenant from request
2. authenticate and authorize user for that tenant
3. open a DB transaction
4. set tenant and global role context in that transaction
5. run tenant-scoped queries using the same transaction client

Use `SET LOCAL` inside a transaction to avoid context leakage across pooled connections:

```sql
SET LOCAL app.current_tenant = 'tenant-uuid-here';
SET LOCAL app.current_global_role = 'Member';
```

**What it does:**

- automatic query scoping
- protection against accidental data leaks

### Deny-by-default RLS setup (no tenant context = no rows)

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

### Sysadmin access without disabling RLS

For tables where platform sysadmins must read across tenants, add a role-aware condition.

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

### Mixed visibility tables (tenant + global rows)

Tables like `event`, `event_participation`, and `award` contain a mix of tenant-scoped and global rows.

**Read policy** — anyone can see global rows; only own-tenant rows otherwise:

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

**Write policy for `event`** — only the owning tenant may create or modify an event:

```sql
CREATE POLICY event_write ON event
FOR INSERT, UPDATE, DELETE
WITH CHECK (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND tenant_id = current_setting('app.current_tenant', true)::uuid
);
```

**Cross-tenant participation writes for global events** — an organizer from any tenant must be able to register or update a participant from a different tenant when the event is `GLOBAL`:

```sql
CREATE POLICY participation_select ON event_participation
FOR SELECT
USING (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND (
    tenant_id = current_setting('app.current_tenant', true)::uuid
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
    tenant_id = current_setting('app.current_tenant', true)::uuid
    OR
    -- global event: any authenticated tenant may write participation
    EXISTS (
      SELECT 1 FROM event e WHERE e.id = event_id AND e.scope = 'GLOBAL'
    )
  )
);
```

**Cross-tenant award writes for global events** — same pattern; issuing tenant may differ from the participant's home tenant:

```sql
CREATE POLICY award_select ON award
FOR SELECT
USING (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND (
    tenant_id = current_setting('app.current_tenant', true)::uuid
    OR EXISTS (
      SELECT 1 FROM event e WHERE e.id = event_id AND e.scope = 'GLOBAL'
    )
  )
);

CREATE POLICY award_write ON award
FOR INSERT, UPDATE
WITH CHECK (
  current_setting('app.current_tenant', true) IS NOT NULL
  AND (
    tenant_id = current_setting('app.current_tenant', true)::uuid
    OR EXISTS (
      SELECT 1 FROM event e WHERE e.id = event_id AND e.scope = 'GLOBAL'
    )
  )
);
```

> The `EXISTS` subquery on `event.scope` is the single source of truth — no application layer needs to duplicate this check for cross-tenant writes.

### Coverage scope

Apply table-specific policy variants across the current model:

- strict tenant tables: `membership`, `membership_role`, `teacher_student_assignment`, `lesson_plan`, `lesson_plan_assignment`, `report`
- mixed visibility tables: `event`, `event_participation`, `award`
- global identity tables (`user`, `teacher_details`, `student_details`) should use explicit policies based on `user_id` ownership and admin role, not tenant_id matching

### Request-to-query sequence

`request -> tenant resolve -> authz -> transaction start -> SET LOCAL app.current_tenant -> tenant query`

This sequence guarantees that every tenant-scoped query is executed with tenant context in the same DB transaction.

### 2. Application-Level Guardrails

```typescript
const tenantPrisma = (tenantId: string) =>
  prisma.$extends({
    query: {
      $allModels: {
        async findMany({ args, query }) {
          args.where = { ...args.where, tenantId };
          return query(args);
        },
        async findFirst({ args, query }) {
          args.where = { ...args.where, tenantId };
          return query(args);
        },
      },
    },
  });
```

### Reusable wrapper do setting db context

```typescript
async function withTenantContext<T>(
  prisma: PrismaClient,
  tenantId: string,
  fn: (tx: Prisma.TransactionClient) => Promise<T>,
): Promise<T> {
  return prisma.$transaction(async (tx) => {
    await tx.$executeRaw`SET LOCAL app.current_tenant = ${tenantId}`;
    return fn(tx);
  });
}
```

## ⚖️ Data isolation approaches Comparison

### 🟡 Schema per Tenant

**Pros:**

- stronger logical isolation

**Cons:**

- complex migrations
- more difficult backups
- poor ORM support
- more complex cross-tenant queries
- noisy neighbor

### 🔵 Database per Tenant

**Pros:**

- maximum isolation
- strong businesses popularity
- no noisy neighbor

**Cons:**

- high infrastructure complexity
- very expensive
- difficult cross-tenant collaboration
