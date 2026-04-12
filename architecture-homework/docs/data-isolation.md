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
If we use multiple databases and need to fetch a specific event participants, we will need to list all dbs, conduct fetching queries for each of them, and finally merge results in application level. In out shared db case we fetch all data with a simple request.

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
- no idle infrastructure per tenant (if we use multiple dbs, we might need to pay for each even if data is not used and activity is minimal)

## 🛡️ Isolation Mechanisms

### 1. Row-Level Security (Primary Enforcement)

Tables must enforce enforce RLS:

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

Each database connection must set:

```sql
SET app.current_tenant = 'tenant-uuid-here';
```

### When to set tenant context

Set tenant context when a request (or background job) starts database work, not just in HTTP middleware.

Recommended order:

1. resolve tenant from request (subdomain/header/token)
2. authenticate and authorize user for that tenant
3. open a DB transaction
4. set tenant context in that transaction
5. run tenant-scoped queries using the same transaction client

Use `SET LOCAL` inside a transaction to avoid context leakage across pooled connections:

```sql
SET LOCAL app.current_tenant = 'tenant-uuid-here';
```

**What it does:**

- automatic query scoping
- protection against accidental data leaks

### Deny-by-default RLS setup (no tenant context = no rows)

Use RLS policies that only allow rows when tenant context is present and matches the row tenant.

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE projects FORCE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS tenant_isolation ON projects;

CREATE POLICY tenant_isolation ON projects
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

Why this is deny-by-default:

- `current_setting('app.current_tenant', true)` returns `NULL` when unset
- policy condition becomes false, so `SELECT/UPDATE/DELETE` see zero rows
- `WITH CHECK` blocks `INSERT/UPDATE` when tenant context is missing or mismatched

Additional hardening:

- ensure your app DB role does **not** have `BYPASSRLS`
- avoid using table owner/superuser for normal app queries
- keep using `SET LOCAL app.current_tenant = ...` inside transactions

### Coverage scope

This policy pattern is applied to **all tenant-scoped tables** (not only `projects`): `users`, `courses`, `events_participation`, `submissions`, and any table containing tenant-owned data.

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

### Reusable wrapper (do not repeat per endpoint)

Do not add `SET LOCAL` manually in every handler. Implement once as a shared helper and call it from services.

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

Usage pattern:

- middleware: resolve tenant and auth
- service: call `withTenantContext(tenantId, ...)`
- repositories: use the provided transaction client (`tx`)

### Verification checklist

- **Test 1 (no context):** without `app.current_tenant`, tenant table queries return zero rows
- **Test 2 (cross-tenant read):** tenant A cannot read rows owned by tenant B
- **Test 3 (cross-tenant write):** `INSERT/UPDATE` with mismatched `tenant_id` is blocked by `WITH CHECK`

## 🔄 Cross-Tenant Collaboration Model

### Key Principle

Cross-tenant access is designed, but not accidental

### Example Data Model

**Event**

- `ownerTenantId` (nullable → global event)

**Participation**

- `tenantId` (participant's school)

### Behavior

- events can be shared across tenants
- participation remains tenant-scoped
- data access is controlled and intentional

## ⚖️ Comparison with Alternatives

### 🟡 Schema per Tenant

**Pros:**

- stronger logical isolation

**Cons:**

- complex migrations
- more difficult backups
- poor ORM support
- difficult cross-tenant queries

### 🔵 Database per Tenant

**Pros:**

- maximum isolation
- strong businesses popularity
- no noisy neighbor

**Cons:**

- high infrastructure complexity
- very expensive
- difficult cross-tenant collaboration
