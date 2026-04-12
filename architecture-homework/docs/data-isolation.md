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

---

## ⚠️ Risks & Mitigations

### 1. Noisy Neighbor

**Risk:** A single tenant running heavy queries (e.g., bulk report exports, large event imports) can exhaust shared database resources — CPU, I/O, connection pool — and degrade performance for all other tenants.

**Mitigations:**

- **Query timeouts** — set `statement_timeout` per connection to cap runaway queries:
  ```sql
  SET statement_timeout = '30s';
  ```
- **Connection pooling limits** — enforce per-tenant connection caps via PgBouncer or similar, preventing one tenant from exhausting the pool
- **Rate limiting at API layer** — throttle expensive endpoints (reports, exports) per tenant using a token-bucket or sliding-window limiter
- **Background job offloading** — move heavy operations (CSV exports, bulk inserts) to async workers (e.g., BullMQ) so they don't compete with real-time requests
- **Resource quotas (future)** — track per-tenant query cost metrics and alert or throttle tenants that exceed thresholds
- **Read replicas** — route reporting/analytics queries to a read replica to isolate OLAP load from OLTP traffic

---

### 2. Scaling — Horizontal vs. Vertical

**Risk:** As tenant count and data volume grow, a single shared database becomes a bottleneck. Choosing the wrong scaling strategy leads to either wasted cost or an unplanned migration under load.

#### Vertical Scaling (Scale Up)

Increase the resources of the existing database instance (CPU, RAM, storage, IOPS).

**When to use:** early growth stage; straightforward to apply with zero schema changes.

**Limits:** has a hard ceiling; becomes expensive at high tiers; single point of failure remains.

**Mitigations:**

- monitor CPU/memory utilisation and upgrade proactively before hitting limits
- use managed cloud databases (AWS RDS, Supabase) to resize with minimal downtime

#### Horizontal Scaling (Scale Out)

Distribute load across multiple nodes.

**Options for this architecture:**

| Approach                   | Description                                                                         | Effort |
| -------------------------- | ----------------------------------------------------------------------------------- | ------ |
| **Read replicas**          | Route read-heavy traffic (reports, dashboards) to replicas                          | Low    |
| **Connection pooling**     | PgBouncer in front of PostgreSQL reduces connection overhead                        | Low    |
| **Caching layer**          | Redis cache for frequently-read, tenant-scoped data                                 | Medium |
| **Tenant sharding**        | Partition tenants across multiple DB instances by `tenantId` hash                   | High   |
| **Move to distributed DB** | Migrate to CockroachDB or Citus (PostgreSQL-compatible) for native horizontal scale | High   |

**Recommended progression:**

1. Start with **read replicas** + **connection pooling** — covers the majority of scale needs with minimal change
2. Add a **Redis cache** for hot-path reads (active users, current events)
3. If a single DB instance becomes a bottleneck at write scale, evaluate **tenant sharding** or a distributed SQL solution
