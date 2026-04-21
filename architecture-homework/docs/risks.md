Additional Global/Systemic Risks

Noisy-neighbor risk in shared database architecture: one high-activity tenant can degrade performance for all tenants.

Proposed solution: performance SLO partitioning as part of platform-level architecture policy.

Tenant-level recovery granularity risk: restoring one tenant cleanly is harder when all tenants share the same database.
Evidence: shared database decision in docs/adr/ADR-002-data-isolation.md#L20 and backup policy in docs/nfr.md#L60.
Proposed solution: Define tenant-scoped recovery strategy and acceptance criteria for partial restore scenarios in disaster recovery planning.

Cross-tenant metadata leakage risk through GLOBAL events: even when row isolation is correct, globally visible records can reveal sensitive organizational activity patterns.
Evidence: global-row visibility in docs/rls-details.md#L81 and docs/rls-details.md#L91, plus cross-tenant collaboration requirement in docs/adr/ADR-002-data-isolation.md#L39.
Proposed solution: Define a global-data classification model that explicitly states which fields are globally visible vs restricted.

Policy complexity risk in mixed-visibility RLS: correctness becomes difficult to reason about as SELECT and write paths diverge for GLOBAL vs TENANT scopes.
Evidence: mixed-table split policy design in docs/adr/ADR-002-data-isolation.md#L30 and detailed participation/award write predicates in docs/rls-details.md#L122 and docs/rls-details.md#L153.
Proposed solution: Treat RLS policy set as a versioned security artifact with scenario-based policy verification as a release gate.

Authorization-layer divergence risk at scale: RBAC/ABAC middleware and RLS can drift semantically over time, causing inconsistent allow/deny behavior across endpoints.
Evidence: layered authorization model in docs/rbac.md#L106, docs/rbac.md#L114, docs/rbac.md#L142 and separate DB policy layer in docs/rls-details.md#L41.
Proposed solution: Define a single authorization semantics catalog that maps endpoint intent to API checks and DB policy expectations.

Global auth hot-path dependency risk: per-request revoked token checks can become a latency and availability bottleneck as platform traffic grows.
Evidence: denylist check for every request in diagrams/auth-sequence.mmd#L76 and docs/security.md#L32.
Proposed solution: Define token validation performance budgets and failure-mode behavior for denylist dependency degradation.

Deployment blast-radius risk: missing deployment topology leaves unclear whether failures are isolated or platform-wide.
Evidence: deployment doc is empty in docs/deployment.md and high-availability assumptions exist in docs/nfr.md#L18 and docs/nfr.md#L67.
Proposed solution: Define platform topology boundaries (region, network, runtime, data plane) and failure domains in deployment architecture documentation.

Compliance and data-governance risk for cross-tenant student data flows: platform supports cross-school participation, but governance boundaries for personal data visibility are not explicit.
Evidence: cross-tenant collaboration in docs/README.md#L14, student/participation entities in docs/data-model.md#L63 and docs/data-model.md#L169, and audit logging requirements in docs/security.md#L68.
Proposed solution: Define a governance matrix for personal data processing by role, tenant boundary, and event scope.
