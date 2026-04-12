# 📦 Data Isolation Strategy

## Overview

This system uses a **shared database multi-tenancy model** with strict isolation enforced at the database level.

Each tenant (school) is logically isolated using a `tenantId` (`schoolId`) while still enabling **controlled cross-tenant collaboration** (e.g., webinars, competitions).

---

## 🏗️ Architecture Choice

### ✅ Shared Database + Row-Level Security (RLS) + Application Guardrails

- Single PostgreSQL database
- Shared tables across all tenants
- Logical isolation via `tenantId`
- Database-level enforcement using Row-Level Security (RLS)
- Application-level safeguards for additional protection

---

## 🎯 Why This Approach Was Chosen

### 1. Product Requirements Fit

The system requires **cross-tenant features**:

- multi-school webinars
- competitions
- shared events

A shared database:

- enables simple queries
- avoids distributed system complexity
- allows clean participation modeling

---

### 2. Fast Development & Iteration

- single schema
- one migration pipeline
- minimal infrastructure complexity
- works seamlessly with Prisma ORM

Ideal for:

- early-stage product
- evolving domain
- rapid feature development

---

### 3. Operational Simplicity

- centralized monitoring
- simpler backups
- easier maintenance
- straightforward tenant onboarding

---

### 4. Cost Efficiency

- one database instance
- efficient resource usage
- no idle infrastructure per tenant

---

### 5. Future Flexibility

The system can evolve into:

- hybrid multi-tenancy
- dedicated databases for tenants

---

## 🔐 Security Model

### Core Principle

Tenant isolation is a **critical security requirement**, not a convenience.

Even if data is not highly sensitive, **any cross-tenant data leak is unacceptable**.

---

## 🛡️ Isolation Mechanisms

### 1. Row-Level Security (Primary Enforcement)

All tenant-bound tables enforce RLS:

```sql
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

Each database connection must set:

```sql
SET app.current_tenant = 'tenant-uuid-here';
```

**Guarantees:**

- automatic query scoping
- protection against accidental data leaks
- enforcement at the lowest level (database)

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

**Purpose:**

- defense-in-depth
- reduces developer mistakes
- ensures consistent filtering

### 3. Mandatory Rules

- RLS must be enabled on all tenant-scoped tables
- Every DB connection must set `app.current_tenant`
- No raw queries without tenant context
- No bypass of RLS policies
- `tenantId` is required on all tenant-owned data

---

## 🔄 Cross-Tenant Collaboration Model

### Key Principle

Cross-tenant access must be explicitly designed, never accidental.

### Example Data Model

**Event**

- `ownerTenantId` (nullable → global event)

**Participation**

- `tenantId` (participant's school)

### Behavior

- events can be shared across tenants
- participation remains tenant-scoped
- data access is controlled and intentional

---

## 📊 Auditability (Future Enhancement)

To improve observability and trust:

- log all data access
- track tenant context per request
- record user actions
- enable audit trails for debugging and compliance

---

## ⚖️ Comparison with Alternatives

### 🟢 Shared Database (Chosen)

**Pros:**

- fastest development
- simplest architecture
- supports cross-tenant features
- low cost

**Cons:**

- requires strict discipline (RLS)
- potential performance contention at scale

### 🟡 Schema per Tenant

**Pros:**

- stronger logical isolation

**Cons:**

- complex migrations
- poor ORM support
- difficult cross-tenant queries
- higher operational overhead

### 🔵 Database per Tenant

**Pros:**

- maximum isolation
- strong enterprise appeal
- performance isolation

**Cons:**

- high infrastructure complexity
- expensive
- difficult cross-tenant collaboration
- complex analytics and reporting
