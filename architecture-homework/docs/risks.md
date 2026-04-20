Risk Analysis (no improvements, only risks)

Authorization contract drift risk across docs can cause incorrect access control behavior at implementation time.
Evidence: docs/api-design.md#L283, docs/api-design.md#L343, docs/api-design.md#L401, docs/api-design.md#L499 vs docs/rbac.md#L63, docs/rbac.md#L66, docs/rbac.md#L68, docs/rbac.md#L96, docs/rbac.md#L172.
Proposed solution: Define one canonical authorization matrix (endpoint -> allowed roles + ABAC checks) and treat all other docs as generated or synchronized views of it.

API endpoint naming drift risk can break frontend-backend integration and test automation.
Evidence: complete-video-upload in docs/api-design.md#L47 vs complete-upload in docs/api-design.md#L114, docs/api-design.md#L492, docs/rbac.md#L70.
Proposed solution: Establish a single API contract source (OpenAPI/typed route map) and align endpoint names in all documents and clients to that source.

Tenant-context source ambiguity risk between client-provided tenant context and JWT-resolved tenant context can lead to inconsistent enforcement paths.
Evidence: frontend sends tenant id from state in docs/front.md#L4, docs/front.md#L103 while backend decision is JWT-first in docs/adr/ADR-001-tenant-resolution.md#L7, docs/adr/ADR-001-tenant-resolution.md#L10, docs/adr/ADR-001-tenant-resolution.md#L15.
Proposed solution: Explicitly state one authoritative tenant-context source for authorization decisions and document client tenant hints as non-authoritative metadata.

Reporting model ambiguity risk: report is described as computed/not stored, but also saved and explicitly modeled as persisted snapshot data.
Evidence: docs/context.md#L100, docs/context.md#L114, docs/context.md#L118, docs/data-model.md#L254, docs/data-model.md#L266.
Proposed solution: Choose one report lifecycle model (purely computed or computed-and-snapshotted) and use the same wording consistently across context, API, and data-model docs.

Cookie policy/config mismatch risk can create environment-dependent authentication failures.
Evidence: refresh cookie path in docs/auth.md#L15 vs docs/security.md#L26, plus strict same-site policy in docs/auth.md#L14 and docs/security.md#L25.
Proposed solution: Consolidate refresh-cookie attributes into one normative configuration block and ensure all docs reference the same values.

Identity model risk: one user can belong to exactly one tenant, which can create account duplication pressure for cross-school participants.
Evidence: docs/data-model.md#L76, docs/data-model.md#L81, while cross-tenant collaboration is central in docs/README.md#L14 and docs/adr/ADR-002-data-isolation.md#L11.
Proposed solution: Record a clear identity policy decision for cross-tenant users (single identity with multi-tenant memberships vs separate identities) and reflect it in data model and flows.

Global-event data attribution risk: global events are tenant-null, while participation/awards remain tenant-bound with cross-tenant writes, increasing semantic complexity around ownership/audit narratives.
Evidence: docs/data-model.md#L151, docs/data-model.md#L152, docs/data-model.md#L176, docs/data-model.md#L209, docs/rls-details.md#L107, docs/rls-details.md#L138, diagrams/tenant-isolation.mmd#L43.
Proposed solution: Define an explicit attribution model (organizing tenant, participant tenant, issuing tenant) and keep field semantics/audit terminology consistent across schema and policy docs.

Scalability direction conflict risk: shared-db RLS decision and NFR mention of per-tenant sharding can pull implementation in competing directions.
Evidence: shared-db decision in docs/adr/ADR-002-data-isolation.md#L20, per-tenant sharding mention in docs/nfr.md#L72.
Proposed solution: Declare the current primary scaling path and frame alternative paths as conditional future states with explicit triggers.

Operational-readiness risk: deployment architecture and runtime topology are unspecified.
Evidence: empty deployment document docs/deployment.md.
Proposed solution: Add a deployment baseline that names runtime topology, environments, networking boundaries, and operational ownership.

Documentation parsing/interpretation risk due to malformed blocks and inconsistent detail depth, which can create misreads during handoff.
Evidence: broken code-fence structure in docs/auth.md#L21 and docs/auth.md#L101, plus mixed granularity across docs.
Proposed solution: Apply a documentation quality pass with markdown linting and a lightweight template for required sections/format depth.

Additional Global/Systemic Risks

Noisy-neighbor risk in shared database architecture: one high-activity tenant can degrade performance for all tenants.
Evidence: one shared PostgreSQL is a core decision in docs/adr/ADR-002-data-isolation.md#L20 and docs/adr/ADR-002-data-isolation.md#L58, while global collaboration and mixed workloads are expected in docs/README.md#L14 and docs/rls-details.md#L79.
Proposed solution: Define explicit tenant fairness controls (resource budgeting, query governance, and performance SLO partitioning) as part of platform-level architecture policy.

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
