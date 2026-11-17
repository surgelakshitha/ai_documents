---
keywords: auth, authentication, login, logout, signup, password, passwords, mfa, oauth, oidc, pkce, keycloak, token, tokens, jwt, session, sessions, cookie, cookies, csrf, cors, refresh token, auth_keycloak
always: false
---

# Security: authentication and sessions

*Part of SECURITY.MD, split by topic (start file: `security.md`). § numbers are those of SECURITY.MD. Binding.*

## 3. Authentication

### 3.1 Protocol choice

- Prefer OAuth 2.0 + OpenID Connect. Public clients (SPA, mobile) MUST use Authorization Code + PKCE. Never implicit flow. Never ROPC.
- Rolling your own auth requires explicit human sign-off. If asked to build a bespoke login, say so first.

### 3.2 Passwords

- Hash with argon2id. Defaults: `memoryCost ≥ 19456 KiB`, `timeCost ≥ 2`, `parallelism 1`. Legacy bcrypt: cost ≥ 12.
- Unique salt per password (the library does this; never supply your own reused salt).
- Verify with the library's `verify()` — it is constant-time. NEVER compare hashes with `===`.

```js
import argon2 from 'argon2';

// store
const hash = await argon2.hash(password, { type: argon2.argon2id });

// verify
const ok = await argon2.verify(hash, password); // never ===
```

> **DO NOT implement client-side password hashing.** The source guideline permits it; it is wrong and this file overrides it. If the browser sends a hash and the server compares it directly, the hash is the password and a database leak becomes a login. TLS already protects the password in transit. If a human insists, the server must still treat the received value as a raw secret and argon2 it — and you must say this out loud.

### 3.3 Password policy

- Minimum length 12. Maximum at least 64 (never truncate silently).
- Check against breach corpora using the HaveIBeenPwned k-anonymity range API (send the first 5 hex chars of the SHA-1 only). NEVER send the password or its full hash off-box. Mandatory for public-facing flows.
- Do not enforce character-composition rules and do not force periodic rotation. Both are deprecated by NIST SP 800-63B and push users toward weaker, predictable passwords. Rotate only on evidence of compromise. (This overrides the source doc.)

### 3.4 Account validation

- Registration/provisioning MUST include a verification step (email confirmation or admin approval) before the account is usable.
- SSO accounts MUST be linked to the internal user model, and the IdP-provided email MUST be verified. If the IdP is not fully trusted (any consumer IdP, any customer-controlled IdP), perform your own email verification. NEVER auto-merge accounts on matching email alone — that is an account-takeover primitive.

### 3.5 MFA

- Support TOTP or WebAuthn. SMS/email OTP only as fallback.
- MFA is mandatory for admin/privileged roles and for every destructive action (account deletion, role grant, key rotation, bulk export). Prompt for the factor immediately before the action, not just at login.

### 3.6 Brute force / credential stuffing

- Rate limit per-IP and per-account on all auth endpoints. Add progressive delay or CAPTCHA after a threshold.
- Login failures MUST return an identical, generic response and take similar time regardless of whether the account exists. Same for password reset and signup. No user enumeration.
- Alert on failed-login spikes, MFA failure spikes, and geo-velocity anomalies.

### 3.7 Tokens

- Sign with RS256 or EdDSA for anything crossing a service boundary. HS256 is forbidden across services — a shared verification secret means any verifier can mint tokens.
- Pin the algorithm on verification. NEVER trust the `alg` header. NEVER accept `none`.
- Validate `exp`, `iat`/`nbf`, `iss`, and `aud` on every verification. A token minted for service A MUST NOT verify at service B.
- Access tokens ≤ 15 minutes. Refresh tokens rotate on use, with reuse detection — a replayed refresh token revokes the whole family.
- Keys live in KMS, are rotated on schedule and on suspected compromise, and are published via JWKS with `kid`.

### 3.8 Recovery, reset, logout

- Reset links: single-use, expire in 10–30 minutes, high-entropy (≥ 128 bits from `crypto.randomBytes`), stored hashed, invalidated on use.
- Reset and password change MUST invalidate all existing sessions and refresh tokens.
- Logout MUST clear the cookie, revoke the refresh token server-side, and propagate to the IdP if OIDC. A logout that only deletes a client-side value is not a logout.
- Users can list and revoke their active sessions/devices.

## 11. Sessions

- Prefer short-lived signed tokens (§3.7) with rotating refresh tokens. If server-side sessions are used, store them in Redis or a dedicated DB table — never the filesystem, never language-native serialization.
- Cookie attributes are N11. Non-negotiable.
- Idle expiry 15–30 minutes; absolute expiry regardless of activity.
- Regenerate the session identifier on login and on privilege elevation (prevents fixation).
- Invalidate all sessions on: logout, password reset, MFA change, role/permission change, account compromise.
- **CSRF:** if any state-changing request authenticates via cookie, you MUST have CSRF protection — `SameSite=Strict` plus a double-submit or synchronizer token. `SameSite` alone is not sufficient for `Lax` flows. Bearer-token-in-header APIs with no cookie auth are exempt; verify that's actually true before claiming it.
- **CORS:** explicit origin allowlist. NEVER reflect the `Origin` header, never `*` with `credentials: true`.
- **Do not bind sessions to IP addresses.** The source doc recommends it; it breaks mobile and CGNAT users, produces constant spurious logouts, and provides little value against an attacker on the same network path. Bind to nothing; detect anomalies through logging and refresh-token reuse detection instead. (This overrides the source doc.)
