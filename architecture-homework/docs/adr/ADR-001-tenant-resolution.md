## Context

The platform is multi-tenant and must guarantee that each request is executed in the correct tenant context.

## Decision

Adopt JWT-first tenant resolution and transaction-scoped tenant context:

1. Extract and validate JWT.
2. Read `tenantId` and user identifier from token payload.
3. Load user and authorize user access to the resolved tenant.
4. Attach user to request object.
5. Execute tenant-scoped DB logic only via `withTenantContext(...)`.
6. Inside transaction set:
   - `SET LOCAL app.current_tenant = <tenantId>`
   - `SET LOCAL app.current_global_role = <role>`
7. Run all repository calls with the same transaction client.

## Diagram

```mermaid
sequenceDiagram
	autonumber
	participant Client
	participant API as Backend API
	participant Auth as JWT Validator
	participant DB as PostgreSQL

	Client->>API: Request + JWT
	API->>Auth: Validate JWT
	Auth-->>API: Claims (userId, tenantId, globalRole)

	alt Invalid token or missing tenantId
		API-->>Client: 401 Unauthorized
	else Valid claims
		API->>DB: Load user + tenant membership
		DB-->>API: User and access result

		alt Not authorized for tenant
			API-->>Client: 403 Forbidden
		else Authorized
			API->>DB: BEGIN
			API->>DB: SET LOCAL app.current_tenant
			API->>DB: SET LOCAL app.current_global_role
			API->>DB: Tenant-scoped queries via tx
			DB-->>API: Scoped rows
			API->>DB: COMMIT
			API-->>Client: 200 OK
		end
	end
```

## Consequences

### Positive

- easy extraction of tenantId
- lower risk of cross-tenant leakage
- consistent behavior across all modules as tenantId is extracted in the middleware

### Negative

- token theft risk: a stolen valid token can be replayed until it expires
- revocation complexity: immediate logout or access removal is harder with stateless tokens
- operational burden: key rotation and strict JWT validation configuration are mandatory

## Alternatives Considered

- Pass `tenantId` from request metadata (`headers` or `req.body`) and trust the client-provided value.
  Rejected: tenant context becomes user-input driven, increases tampering risk.