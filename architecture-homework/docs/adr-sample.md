5 Decisions You Already Made (ADR Candidates, not writing ADRs)

You chose a modular monolith backend with CQRS-lite and strategy-based domain variation.
Evidence: docs/containers.md#L47, docs/containers.md#L48, docs/containers.md#L49, docs/containers.md#L100.

You chose JWT-first tenant resolution with transaction-scoped DB context for every tenant-scoped operation.
Evidence: docs/adr/ADR-001-tenant-resolution.md#L7, docs/adr/ADR-001-tenant-resolution.md#L13, docs/adr/ADR-001-tenant-resolution.md#L15.

You chose shared PostgreSQL plus FORCE RLS as the primary tenant isolation mechanism, including mixed-visibility policy design for global events.
Evidence: docs/adr/ADR-002-data-isolation.md#L20, docs/adr/ADR-002-data-isolation.md#L22, docs/adr/ADR-002-data-isolation.md#L30, docs/rls-details.md#L41.

You chose layered authorization: route-level RBAC, route-specific ABAC checks, and DB-level RLS as separate control layers.
Evidence: docs/rbac.md#L106, docs/rbac.md#L114, docs/rbac.md#L142.

You chose a token/session model with in-memory access token, HttpOnly refresh cookie rotation, and denylist revocation checks.
Evidence: docs/auth.md#L35, docs/auth.md#L63, docs/auth.md#L75, docs/auth.md#L96, diagrams/auth-sequence.mmd#L50, diagrams/auth-sequence.mmd#L78.

postgres, magic link 2fa, jwt, sth else

react over next.js
