---
keywords: security review, security checklist, self-check, security self-check, deviation, deviations, reject on sight, fast greps
always: false
---

# Security: self-check, fast greps and deviations

*Part of SECURITY.MD, split by topic (start file: `security.md`). § numbers are those of SECURITY.MD. Binding.*

## 15. Self-Check — output this before you finish

Answer every line. "N/A" is acceptable only with a reason.

```
[ ] Inputs: schema-validated at the boundary, unknown keys rejected
[ ] AuthZ: guard/policy enforced server-side; ownership checked, not just role
[ ] Tenancy: every query/cache key/job/storage path scoped by tenant_id
[ ] Secrets: none introduced in code, config, tests, or logs
[ ] Injection: no string-concatenated SQL / shell / path / HTML
[ ] Output: no internal detail in client responses; generic error + trace ID
[ ] Logging: no credentials, tokens, or PII; security events logged
[ ] Crypto: argon2id / AES-256-GCM / crypto.randomBytes only
[ ] Sessions: cookie flags correct; invalidation paths covered; CSRF handled
[ ] Network: no new outbound call to a user-controlled URL; no new public exposure
[ ] Dependencies: none added (or: named and justified below)
[ ] Controls: none disabled, weakened, or bypassed
[ ] Uncertainties flagged for human review: __________
```

## 16. Fast greps — reject on sight

If your diff introduces any of these, stop and reconsider:

```
Math.random()                       innerHTML =
eval(                               dangerouslySetInnerHTML
new Function(                       v-html
child_process.exec(                 sslmode=disable
`SELECT ... ${ }`                   rejectUnauthorized: false
md5( / sha1( / sha256(password      NODE_TLS_REJECT_UNAUTHORIZED=0
localStorage.setItem('token'        Access-Control-Allow-Origin: *
process.env.X || 'hardcoded'        verify(token) without algorithms: [...]
jwt.decode(                         catch (e) {}
.merge( on untrusted input          0.0.0.0/0 in a security group
```

## 17. Deviations from Secure Coding Guidelines v1.0.0

This file intentionally departs from the source in six places. Each is a case where following the source literally produces weaker security. Escalate to the security owner if you disagree — do not resolve it in code.

| # | Source | This file | Why |
|---|--------|-----------|-----|
| 1 | §2.3 client-side password hashing | Forbidden | The transmitted hash becomes the credential; a DB leak becomes direct login. |
| 2 | §5.2 HEAD-then-GET URL validation | Replaced with isolate + resolve-and-pin + no-redirect | HEAD is itself SSRF and is trivially bypassed by rebinding/redirect (TOCTOU). |
| 3 | §12.7 bind session to IP | Forbidden | Breaks mobile/CGNAT, causes spurious logouts, weak against same-path attackers. |
| 4 | §2.8 composition rules + rotation | Length + breach check only | Deprecated by NIST SP 800-63B; drives predictable passwords. |
| 5 | §2.9 HS256 permitted | Asymmetric required cross-service | Shared HMAC secret lets any verifier forge tokens. |
| 6 | §11.4 AES-256-CBC + HMAC permitted | GCM only | Encrypt-then-MAC is implemented wrong more often than right. |

**Gaps in the source now covered here:** CSRF, CORS, ReDoS, prototype pollution, path traversal, command injection, open redirect, mass assignment, IDOR/ID enumeration, secure randomness, dependency confusion, unsafe deserialization, SVG upload risk, and existence-leaking status codes.
