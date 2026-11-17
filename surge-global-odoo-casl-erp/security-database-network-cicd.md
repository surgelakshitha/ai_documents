---
keywords: sql, query, queries, database, postgres, postgresql, orm, migration, migrations, cursor, execute, tls, ssl, network, egress, service-to-service, mtls, dependency, dependencies, requirements.txt, package.json, lockfile, ci, cd, pipeline, docker, dockerfile, deployment, deploy, env, odoo.conf, api key, kms, backup
always: false
---

# Security: database, services, supply chain and CI/CD

*Part of SECURITY.MD, split by topic (start file: `security.md`). § numbers are those of SECURITY.MD. Binding.*

## 12. Database

- Per-service least-privilege user (N15). Migration/DDL credentials are separate and never used at runtime.
- Never grant application users: `DROP`, `TRUNCATE`, `ALTER` on production tables, `GRANT`, `SUPERUSER`, or system catalog access.
- Private subnets only (N16). TLS enforced at the server (`sslmode=verify-full` client-side; reject unencrypted connections server-side). Rotate credentials on schedule and on suspicion.
- Parameterized queries or prepared statements, always. ORM query builders are fine; ORM raw-SQL escape hatches with interpolated values are not.

```js
// NEVER
db.query(`SELECT * FROM invoices WHERE id = '${id}'`);
// ALWAYS
db.query('SELECT * FROM invoices WHERE id = $1 AND tenant_id = $2', [id, tenantId]);
```

- Identifiers (table/column names, `ORDER BY` targets) cannot be parameterized — map user input through an allowlist. Never interpolate them.
- Query timeouts and connection pool limits set explicitly. An unbounded query is a DoS.
- External access (partners, integrations) goes through an API layer or a read-only replica — never a direct connection to primary.
- Audit logging on: auth attempts, DDL, privileged operations, sensitive-table access. Ship to the central stack.
- Backups encrypted, stored separately, access-restricted, restoration tested, PITR enabled. An untested backup is not a backup.

## 13. Service-to-Service

- mTLS between services, or TLS 1.2+ with signed, audience-restricted tokens. Zero trust: compromise of one service must not grant access to another.
- Service identity is verified on every internal call. "It came from inside the VPC" is not authentication.
- DNS-based discovery, never hardcoded IPs.
- Explicit allow-list network policy: service A can reach only what it needs. Everything else denied. Kubernetes NetworkPolicy / security groups / mesh authz.
- Egress denied by default. A service with no need for internet access gets none.
- Rate limits, timeouts, retries with jittered backoff, and circuit breakers on every internal call. Retry storms are self-inflicted DoS.
- Distributed tracing across all hops, with auth status recorded.

## 14. Supply Chain, Secrets, CI/CD

- Secrets from AWS Secrets Manager / Vault / SSM at runtime. NEVER in code, `.env` committed to git, CI logs, Docker `ENV`, or image layers. Least-privilege access, rotated, audited.
- If you notice a committed secret: stop, tell the user it must be rotated, and do not merely delete it from the working tree — git history retains it.
- `package.json`: pin with a committed lockfile. Vet new dependencies (maintenance, downloads, install scripts). Prefer stdlib over a package for trivial functionality.
- Watch for typosquats and dependency confusion. Scoped internal packages must not be resolvable from the public registry.
- CI runs on every PR: secret scanning, SAST (Semgrep/CodeQL), dependency scan (Dependabot/Snyk/Trivy), license check. Build fails on high/critical CVE.
- Patch SLA: critical 24–72h, high within a week.
- Sign build artifacts (Cosign/Sigstore) and verify signatures at deploy.
- Maintain an SBOM.
- CI credentials are least-privilege, environment-scoped, rotated. PRs from forks never receive secrets.
- NEVER add a new dependency, a new outbound network call, or a new privileged CI step without saying so explicitly in your summary.
