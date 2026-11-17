---
keywords: authorization, permission, permissions, role, roles, access, access rights, acl, guard, policy, casl, ability, tenant, tenants, multi-tenant, ownership, idor, admin, sudo, ir.rule, ir.model.access, record rule, groups, res.groups, portal, privilege
always: false
---

# Security: authorization (CASL) and multi-tenancy

*Part of SECURITY.MD, split by topic (start file: `security.md`). § numbers are those of SECURITY.MD. Binding.*

## 4. Authorization

### 4.1 Enforcement shape

- Deny by default. A route with no explicit policy is a bug that must fail closed.
- Authorization lives in a guard/middleware/policy layer, applied declaratively per endpoint — not scattered `if (user.role === 'admin')` checks in handlers. Duplicated inline checks are how bypasses happen.
- Use CASL (`@casl/ability`) as the policy engine (project decision, §4.5) so the same rule set drives API enforcement and UI affordances. UI uses it to hide; API uses it to deny.

### 4.2 Roles and resources

- Roles map to named permissions. NEVER compare role strings inline.
- Ownership checks are separate from role checks and are also mandatory: "is this user allowed to edit invoices" AND "is this specific invoice theirs". Missing the second is IDOR — the single most common finding in this codebase's class of application.
- Use non-sequential public identifiers (UUIDv4/ULID). Sequential integer IDs in URLs make enumeration trivial. This is defense-in-depth, not a substitute for the ownership check.

### 4.3 Elevated operations

- Admin endpoints require an explicit high-privilege permission, MFA (§3.5), and an audit log entry containing actor, target, action, tenant, timestamp, trace ID.
- NEVER expose an admin capability on a shared endpoint gated only by a request field (`{ "asAdmin": true }`). Separate route, separate policy.

### 4.4 Authorization caching

- Cache authorization decisions no longer than the token lifetime, and only if you have an invalidation path. Role/permission changes MUST take effect within the access-token TTL — that is why the TTL is short.

### 4.5 CASL in this project (project decision)

CASL (`@casl/ability`) is the policy engine for every JavaScript/TypeScript layer that makes or shows an authorization
decision (the orchestrator API, the portals, Odoo OWL front ends). Do not write permission checks beside it.

- **One place defines the rules.** A single `defineAbilityFor(user)` per service builds the ability from the validated
  token: its named permissions and its one active tenant (§5.1). No other file defines rules.
- **Actions and subjects.** Actions are `read`, `create`, `update`, `delete` plus named business actions (`approve`,
  `export`); subjects are model names. `manage` and `all` are reserved for an explicit super-admin permission, and
  every use of it is audit-logged (§4.3).
- **Tenant and ownership are conditions of the rule** (§4.2, §5.2, N5), not checks added later:

```ts
import { AbilityBuilder, createMongoAbility } from '@casl/ability';

export function defineAbilityFor(user: SessionUser) {
  const { can, build } = new AbilityBuilder(createMongoAbility);
  can('read', 'Invoice', { tenantId: user.tenantId });
  can('update', 'Invoice', { tenantId: user.tenantId, ownerId: user.id });
  if (user.permissions.includes('invoice:approve')) can('approve', 'Invoice', { tenantId: user.tenantId });
  return build();   // anything no rule allows is denied
}
```

- **The guard runs before the handler and checks the loaded record**, not only the type:
  `ForbiddenError.from(ability).throwUnlessCan('update', subject('Invoice', invoice))`. A route with no policy is
  denied. Map `ForbiddenError` to 404 or 403 as in §9.
- **Queries are restricted by the ability** (`accessibleBy` of the CASL adapter for the ORM in use) in addition to the
  tenant predicate (§5.2). Never load everything and filter in memory.
- **The UI gets the packed rules** (`packRules`) and uses `ability.can(...)` only to show or hide controls. It is never
  the enforcement.
- **Odoo.** Keep permission names aligned with the Odoo groups so a role means the same in CASL and in Odoo. Odoo keeps
  enforcing with groups, `ir.model.access` and record rules (`ir.rule`). The `api_action_*` methods run as SUPERUSER,
  so each one MUST check authorization inside the method even when CASL already allowed the call upstream. Do not rely
  on the caller.
- **Cache the built ability** per request, or per token within the access-token TTL (§4.4). Never across users or
  tenants.

## 5. Multi-Tenancy

This is the highest-blast-radius category in this platform. A tenant leak is a customer-reportable incident.

### 5.1 Context

- Tenant context comes from a validated source on every request: the `X-Tenant-ID` header cross-checked against the token's tenant claim, or the token claim alone. Missing or mismatched → `403`. NEVER infer tenant from a body field, a referer, or a subdomain alone.
- Tokens are scoped to exactly one tenant. A user with access to five tenants holds a token for one at a time. Switching goes through a dedicated authenticated, audited endpoint (`/auth/switch-tenant`) that issues a new token and invalidates the old one.

### 5.2 Data layer

- Every query includes the tenant predicate. Enforce it where it cannot be forgotten:
  - Postgres RLS with a per-request `SET LOCAL app.tenant_id`, plus a `FORCE ROW LEVEL SECURITY` policy; or
  - a repository/ORM base class that injects the predicate and throws if tenant context is absent.
- NEVER write a raw query in a tenant-scoped table without the predicate, even in a migration, a report, or an admin script.
- Cross-tenant reads are permitted only for an explicitly modelled super-admin permission, and every such read is audit-logged.

### 5.3 Everything else

- Cache keys, queue message keys, S3 prefixes, search indices, rate-limit buckets, and idempotency keys all include the tenant ID. A cache key without a tenant ID is a cross-tenant leak waiting for a hash collision of user input.
- Background jobs carry tenant context explicitly. A job that resolves tenant from "the last request" is broken.
- The UI always displays the active tenant.
