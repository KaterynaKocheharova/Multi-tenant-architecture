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

**What it does:**

- automatic query scoping
- protection against accidental data leaks

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
```
