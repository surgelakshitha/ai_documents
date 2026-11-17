---
keywords: 
always: false
---

# Secure Coding Guidelines v1.0.0

**Surge Global**

> **Reference document.** This is the full text of the source guideline, converted from the PDF without changing its
> wording. The agent-oriented, binding version is SECURITY.MD, split by topic into [security.md](security.md) (start here) and
> the `security-*.md` files, which **overrides this
> document in six places** (its section 17, "Deviations"). Where the two disagree, `security.md` wins. The six
> places are marked below with ⚠.
>
> Too large to send to the AI in one prompt (the per-run document limit is small), so it has no keywords: it is read by
> people and by tools that open files directly.

## Contents

1. Overview
2. Authentication Specification
3. Authorization Specification
4. Multi-Tenancy Specification
5. Image Upload by URL Specification
6. Internal Service Communication Specification
7. Handling Untrusted HTML Specification
8. Handling Untrusted User Inputs
9. Secure Error Handling
10. Logging and Monitoring Specification
11. Password Management Specification
12. Session Management Specification
13. Developer & Team Security Practices
14. Database Security Specification

---

## 1. Overview

This document outlines secure coding practices to be followed during product development. Adhering to these guidelines
ensures that security is embedded into the design and implementation of the product from the outset.

---

## 2. Authentication Specification

### 2.1 General Requirements

- All authentication mechanisms must be implemented securely to prevent unauthorized access and user impersonation.
- Passwords must never be logged in any form in application logs.

### 2.2 Preferred Protocol

- Where possible, authentication should be implemented using OAuth 2.0 with OpenID Connect.
- The PKCE (Proof Key for Code Exchange) extension should be used for public clients to mitigate authorization code
  interception attacks.

### 2.3 Fallback Authentication (Non-OIDC)

If OpenID Connect cannot be implemented, the following minimum requirements must be enforced:

- Passwords must be hashed using a secure, modern algorithm (e.g., bcrypt, argon2) before storing them in the database.
- ⚠ *Overridden by security.md:* If feasible, client-side hashing of passwords should be performed before
  transmission. Only the hashed value should be sent to the backend.
- Ensure that:
  - Logging of any user credentials (raw or hashed) is strictly prohibited.
  - The implementation is hardened against bypass techniques, such as insecure fallback logic or improper session
    validation.

### 2.4 Account Validation

- An account verification step (e.g., email confirmation or admin approval) must be included in the user registration or
  provisioning flow.
- If SSO (Single Sign-On) is used:
  - User accounts must be linked to the platform's internal user model.
  - The email address provided by the SSO identity provider must be verified.
  - If the SSO identity provider is not fully trusted, an additional email verification must be performed before
    granting access.

### 2.5 Session Management

- Sessions must be time-limited and expire after a period of inactivity (e.g., 15–30 minutes).
- Implement refresh tokens for session renewal if long sessions are needed.
- Session identifiers or tokens must be:
  - Cryptographically secure
  - Stored in HttpOnly, Secure, SameSite=strict cookies (if using cookies)
- Invalidate all active sessions on password reset or account compromise.

### 2.6 MFA (Multi-Factor Authentication)

- Support for Multi-Factor Authentication (MFA) should be implemented.
- MFA must be enforced for privileged/admin users or high-risk actions. E.g.: when an account deletion step is
  available, make sure to send the MFA token before the dangerous action is taken.
- Support common MFA mechanisms: TOTP (e.g., Google Authenticator), SMS/Email (less preferred), or WebAuthn.

### 2.7 Brute Force & Credential Stuffing Protections

- Implement rate-limiting, CAPTCHA, or progressive delays on authentication endpoints.
- Use anomaly detection (e.g., IP reputation, geo-velocity) to detect suspicious login attempts.
- Support for login alerts or account lockouts on repeated failures (configurable).

### 2.8 Password Policy

⚠ *Overridden by security.md (length and breach check only; no composition rules, no forced rotation):*

- Enforce strong password requirements:
  - Minimum length (e.g., 12 characters)
  - Require complexity or passphrases
- Disallow common/breached passwords using services like HaveIBeenPwned
- Implement password rotation when applicable

### 2.9 Secure Token Handling

- ⚠ *Overridden by security.md (asymmetric signing required across services):* Tokens (access, refresh, etc.) must be
  signed (e.g., JWT with RS256 or HS256).
- Validate token expiration (`exp`), issuance (`iat`), and audience (`aud`) claims.
- Rotate secrets/keys periodically and on compromise.

### 2.10 Logging & Monitoring

- Log only authentication metadata (e.g., IP, timestamp, user ID), never credentials.
- When logging P1 and P2 data make sure those details are masked.
- Ensure centralized logging with anomaly detection or SIEM integration.
- Alert on:
  - Failed login spikes
  - MFA failures
  - Disabled/bypassed authentication methods

### 2.11 Device and Browser Trust

- Consider implementing device fingerprinting or trusted device recognition.
- Allow users to view and revoke active sessions/devices.

### 2.12 Secure Logout

- Provide a logout endpoint that:
  - Invalidates session tokens and cookies
  - Propagates logout across all SSO sessions if applicable
  - Force logout and invalidate the session after a password reset

### 2.13 Recovery & Reset Flows

- Password reset and account recovery flows must:
  - Require verification (e.g., via email or MFA)
  - Expire recovery links quickly (e.g., 10–30 minutes)
  - Be resistant to enumeration and replay attacks

### 2.14 Breach Protection

- The system should integrate with the Have I Been Pwned API (v3) to check whether a user's chosen password has
  appeared in known data breaches.
- During password creation or reset, the plaintext password must not be transmitted to the external API. Use the Pwned
  Passwords k-Anonymity model to securely check for breached passwords without revealing the full password.
- If a password is found in breach datasets:
  - Reject the password and prompt the user to choose a different one.
  - Optionally, log or monitor repeated breach attempts for user risk profiling.
- This check should be optional for internal tools, but must be enforced for all public-facing user registration and
  password reset flows.

---

## 3. Authorization Specification

### 3.1 General Principles

- Authorization decisions must be enforced server-side. Never rely solely on client-side checks. Implementation should
  be available on guard levels for each endpoint.
- The principle of least privilege must be applied: users and services should be granted only the minimum permissions
  necessary to perform their functions. E.g.: read-only users should not have access to edit resources.
- All access control logic must be centralized or consistently implemented across services to prevent bypasses; this
  should be implemented on a guard level.

### 3.2 Role-Based Access Control (RBAC)

- The system must define a clear role model (e.g., admin, user, moderator, viewer, etc.).
- Roles must be associated with a defined set of permissions or capabilities.
- Authorization checks must validate the requesting user's role against the action being performed or the resource being
  accessed.

### 3.3 Resource-Based Access Control

- For actions involving resource ownership (e.g., document access, project updates), the system must validate:
  - That the authenticated user has access to the specific resource instance.
  - That access rights (e.g., read/write/delete) align with the resource's ACL (Access Control List) or policy.
- Resource access rules should be enforced using a policy engine or middleware layer where applicable.
- This can be achieved by utilizing a library like CASL.js.

### 3.4 Multi-Tenancy Isolation

- In multi-tenant systems:
  - Every request must be scoped to the tenant context (e.g., via tenant ID or domain).
  - Cross-tenant access must be strictly prohibited and validated at the service and data layer unless the user is
    super-admin or an administrative user with permissions across multiple tenants. Even when the user has multiple
    tenant access the token should be scoped to the current authorized tenant.
  - All API and UI components must enforce tenant boundary checks.

### 3.5 Administrative Controls

- Administrative or elevated operations must be accessible only to users with the correct high-privilege role(s).
- High-privilege actions (e.g., user deletion, role assignment) should be logged and monitored.
- UI and API must hide or restrict administrative functions from unauthorized users.

### 3.6 Service-to-Service Authorization

- Internal service communication must use signed tokens or mutual authentication (e.g., mTLS).
- Each internal service must validate the identity and scope of the calling service before granting access.
- Use audience-restricted tokens (e.g., `aud` claim in JWT) to prevent token replay across services.

### 3.7 Authorization Caching & Token Use

- Authorization decisions should not be cached beyond the token lifetime unless invalidation is supported (e.g.,
  revocation lists).
- Use short-lived access tokens with clearly defined scopes and roles.
- Scope definitions must align with resource-level access needs and not be overly broad.

---

## 4. Multi-Tenancy Specification

### 4.1 Tenant Isolation

- The system must enforce strict tenant isolation at the application, API, and data layers.
- All operations, including read, write, update, and delete, must be scoped to a specific tenant.
- Cross-tenant access must be explicitly denied unless elevated privileges or tenant-switching workflows are used.

### 4.2 Tenant Context Enforcement

- Every request must include the tenant context to determine access scope.
- The tenant context must be validated against:
  - The authenticated user's allowed tenants
  - The permissions associated with the current token/session
- Tenant context should be provided via a dedicated HTTP header, e.g., `X-Tenant-ID`.
  - Avoid inferring tenants solely from user ID or token unless encoded securely and validated.

### 4.3 Token Scoping per Tenant

- Authentication tokens (e.g., JWTs) must be scoped to a single tenant.
- Even if a user has access to multiple tenants, the token must only allow actions within one tenant context at a time.
- When scoping a token make sure the token has the required scope only.
- Recommended behavior:
  - Require the user to explicitly switch tenants, which will trigger a new authentication or token issuance process.
  - Alternatively, provide a dedicated endpoint (e.g., `/auth/switch-tenant`) to issue a tenant-scoped token when
    switching between tenants.

### 4.4 UI & API Behavior

- The UI must clearly reflect the current tenant context to avoid confusion or accidental actions in the wrong tenant.
- API clients must explicitly set the tenant header (`X-Tenant-ID`) in all requests.
- If the header is missing or invalid:
  - The request must be rejected with a 400 Bad Request or 403 Forbidden response.

### 4.5 Secure Tenant Switching

- If tenant switching is supported:
  - It must only be allowed for users explicitly granted access to the target tenant.
  - The switching action must be authenticated and audited.
  - The system should invalidate or limit the old token upon issuing a new one for a different tenant.

### 4.6 Database-Level Enforcement

- If multi-tenancy is implemented at the data layer (shared schema or shared DB):
  - Every query must include tenant filtering as part of its logic (e.g., `WHERE tenant_id = ?`).
  - Use row-level security (e.g., PostgreSQL RLS) or enforced query scopes to prevent data leaks.
- Consider using tenant-aware ORMs or service layers to abstract this enforcement.

### 4.7 Logging & Auditing

- All tenant-specific actions must be logged with tenant context (tenant ID, user ID, operation).
- Audit logs should support filtering by tenant and be stored in a segregated or logically partitioned manner.

---

## 5. Image Upload by URL Specification

### 5.1 General Principles

- Allowing users to upload images via external URLs introduces security risks, including SSRF (Server-Side Request
  Forgery). Proper validation and isolation must be enforced to mitigate these risks.

### 5.2 URL Validation Flow

⚠ *Overridden by security.md (isolate the fetcher, resolve and pin the IP, no redirects; a HEAD request is itself an
SSRF):*

- Before fetching any external image via a GET request, the system must first issue a HEAD request to validate the
  target URL.
- The GET request to download the image must only proceed if the HEAD request returns a successful status (e.g.,
  200 OK).
- If the HEAD request fails, the operation must be aborted without exception.

### 5.3 Network Isolation

- If image-fetching by URL is a requirement, the fetching component should be isolated from the core application
  infrastructure.
  - This can be achieved by deploying it in a separate VPC, subnet, or security group that has no access to internal
    services or metadata APIs.
- Recommended implementation:
  - Use a serverless function (e.g., AWS Lambda) to fetch and validate the image.
  - Save the validated image directly to an object store like Amazon S3 using scoped AWS credentials.

### 5.4 Internal IP Blocking

- If isolation in a separate VPC is not possible, the image fetching logic must explicitly block requests to internal IP
  ranges, including:
  - `127.0.0.0/8` (loopback)
  - `169.254.169.254` (AWS metadata service)
  - `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (private networks)
  - `localhost`, `*.internal`, and known internal service domains
- Use libraries such as `http-acl` to enforce IP allow/deny rules.

### 5.5 Additional Security Measures

- Set a maximum allowed file size and timeout for the download operation.
- Limit allowed content types to image MIME types (`image/png`, `image/jpeg`, etc.).
- Ensure proper validation and sanitization of:
  - File extension
  - Content-Type header
  - Actual file content (e.g., using file signature/magic bytes)

### 5.6 Logging and Monitoring

- Log all image-fetching attempts with:
  - Source IP
  - Target URL
  - HTTP response code from HEAD and GET
- Monitor for:
  - Repeated attempts to access internal IPs
  - Excessive failures or timeouts
  - Suspicious URLs or patterns (e.g., access to non-image content)

### 5.7 Pre-Signed URL Uploads

#### 5.7.1 Overview

- Pre-signed URLs provide a secure alternative to server-side URL fetching by allowing clients to upload images
  directly to cloud storage (e.g., AWS S3, Google Cloud Storage, Azure Blob Storage).
- This approach eliminates SSRF risk entirely since the server never fetches from user-supplied URLs.

#### 5.7.2 Security Requirements

- Pre-signed URLs **MUST** expire within 5–15 minutes of generation.
- URLs **SHOULD** be configured for single use where supported by the storage provider.
- URLs **MUST** restrict uploads to allowed MIME types (`image/png`, `image/jpeg`, `image/webp`, `image/gif`).
- URLs **MUST** enforce a maximum file size at the storage level (recommended: 10 MB).
- Upload targets **MUST** be a dedicated bucket/container isolated from sensitive application data.

#### 5.7.3 Upload Workflow

1. Client requests a pre-signed upload URL from the application server.
2. Server authenticates and authorizes the requesting user.
3. Server generates a pre-signed URL with constrained permissions and returns it to the client.
4. Client uploads the image directly to the storage provider.
5. Client notifies the server of upload completion, or the server listens for storage events (e.g., S3 Event
   Notifications).
6. Server performs post-upload validation before making the image accessible.

#### 5.7.4 Post-Upload Validation

Even with pre-signed URL constraints, uploaded files **MUST** be validated before use:

- Verify MIME type matches actual file content using magic bytes.
- Re-check file size against application limits.
- Re-encode the image to strip potentially malicious metadata or embedded payloads.
- Optionally scan for malware using services such as AWS GuardDuty or ClamAV.

#### 5.7.5 Access Control

- Uploaded objects **MUST** be stored with private ACLs by default.
- Images **SHOULD** be served through a CDN or application-controlled endpoint rather than exposing raw storage URLs.
- If direct storage access is required, use separate pre-signed download URLs with short expiration times.

#### 5.7.6 Logging and Monitoring

- Log all pre-signed URL generation requests with:
  - User ID
  - Generated URL identifier or object key
  - Timestamp and expiration time
- Monitor for:
  - Excessive URL generation requests from a single user
  - Failed or abandoned uploads
  - Uploads that fail post-validation checks

---

## 6. Internal Service Communication Specification

### 6.1 Secure Transport

- All internal service-to-service communication must use mutual TLS (mTLS) to:
  - Encrypt communication between services
  - Authenticate both the client and the server
- If mTLS is not feasible, traffic must be protected using at least TLS 1.2+ with trusted certificates.

### 6.2 Service Discovery

- Services must communicate using DNS-based service discovery (e.g., `user-service.internal.local`), rather than
  hardcoded IP addresses.
- This enables:
  - Better fault tolerance
  - Dynamic scaling
  - Easier network isolation and observability

### 6.3 Least-Privilege Network Policies

- Communication between services must be explicitly defined and restricted:
  - Only allow services to talk to the services they require.
  - For example, if `utility-service` does not need to access `user-service`, its access must be denied.
- This can be implemented using:
  - Kubernetes Network Policies
  - Cloud firewall rules (e.g., AWS Security Groups)
  - Service Mesh authorization policies (see Istio Authorization)

### 6.4 Service Mesh Usage

- A service mesh (e.g., Istio, Linkerd) should be used to manage:
  - mTLS between services
  - Fine-grained traffic policies
  - Authorization policies (allow/deny per route, HTTP method, or identity)
  - Observability and auditing of inter-service traffic

### 6.5 Authentication & Authorization Between Services

- Internal APIs must enforce authentication even within the same trust boundary when possible using service accounts.
  - Use JWTs, signed tokens, or mTLS identities to validate calling services.
- Use the `aud` (audience) claim or scoped credentials to ensure that tokens are not reused across unintended services.

### 6.6 Avoid Implicit Trust

- Do not assume that all services within the internal network are trusted:
  - Compromise of one service should not allow lateral movement.
  - Apply zero trust principles even within internal traffic.

### 6.7 Observability & Monitoring

- All internal requests should be:
  - Traced end-to-end using a distributed tracing solution (e.g., OpenTelemetry, Jaeger, Zipkin)
  - Logged with service identifiers, endpoint, latency, and auth status
  - Monitored for anomalies (e.g., unexpected service calls or spike in internal failures)

### 6.8 Rate Limiting and Throttling

- Apply rate limits between services to protect against:
  - Resource exhaustion from faulty or compromised services
  - Unintentional DoS scenarios
- Implement retry policies with backoff and circuit breakers to avoid cascading failures.
- Implement debouncing when required.

### 6.9 Static Egress Control

- If a service does not need outbound internet access, it must be restricted via egress policies.
- Use proxy firewalls or egress gateways to filter and inspect outbound traffic from services.

---

## 7. Handling Untrusted HTML Specification

### 7.1 Overview

When rendering user-generated or third-party HTML content (e.g., blog posts, rich comments), proper sanitization is
mandatory to prevent Cross-Site Scripting (XSS) and HTML injection vulnerabilities.

### 7.2 Mandatory Sanitization

- All untrusted HTML content must be sanitized before rendering it in the browser.
- Use DOMPurify, a widely adopted and actively maintained HTML sanitizer.
  - DOMPurify removes malicious elements such as `<script>`, `onerror`, `javascript:` URLs, etc.
  - It supports both client-side and server-side sanitization (Node.js/DOM environments).
- Sanitization must be applied consistently, especially in the following contexts:
  - Rendering with `dangerouslySetInnerHTML` (React)
  - Assigning content to `.innerHTML` (vanilla JS)
  - Any markdown-to-HTML conversion
  - Embedding HTML inside WYSIWYG editors

### 7.3 Do Not Trust Input at Rest

- Never store already-rendered HTML in the database. Store raw user input (e.g., Markdown or plain text) and sanitize
  only at render time.
- This prevents future bypasses if your sanitizer's rules or versions change.

### 7.4 Stored XSS Defense

- Stored HTML input must be treated the same as reflected HTML: sanitize before use.
- Ensure all APIs returning user content to the front end are treated as potential vectors for stored XSS.

### 7.5 Additional Best Practices

- Strictly limit allowed HTML tags and attributes:
  - Avoid allowing dangerous elements such as `<iframe>`, `<embed>`, `<object>`, `<script>`, `<style>`, etc.
  - Whitelist only safe elements (`<b>`, `<i>`, `<p>`, `<a>`, `<img>`, etc.) as needed.
- For `<img>` tags:
  - Sanitize to remove `onerror`, `onload`, and `src="javascript:"` payloads.
  - Optionally use a proxy or whitelist image domains to prevent tracking or malicious links.

### 7.6 CSP (Content Security Policy)

Implement a strict CSP header as a defense-in-depth mechanism:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none';
```

- Block inline scripts unless absolutely necessary.
- If you must allow rich HTML, use trusted-types to enforce sanitization before `innerHTML` access.

### 7.7 SSRF/Tracking Vector Controls

- Be cautious of users embedding the following if the application does server-side rendering:
  - `<img src="http://localhost">`
  - `<video poster="http://127.0.0.1:8000">`
  - `<iframe src="http://internal-service">`
- DOMPurify removes these by default, but double-check domain access if rendering in trusted environments.

---

## 8. Handling Untrusted User Inputs

### 8.1 Principle of Input Validation

- All user input must be treated as untrusted by default.
- Input validation is mandatory for every entry point, including:
  - API requests
  - Web forms
  - Query parameters
  - Headers and cookies
  - WebSocket messages

### 8.2 Validation Rules

- Input must be validated strictly based on expected type and format:
  - If an input is expected to be a UUID, only allow valid UUID formats (e.g., v4 UUID regex).
  - If an input should be a number, reject any strings or mixed types.
  - If the input is an enum, accept only known, pre-defined values.
- Never trust client-side validations. All validation must be enforced on the server side.

### 8.3 Validation Libraries

- Use a well-maintained validation library (e.g., Joi, Zod, Yup) to:
  - Define strict schemas for request payloads
  - Block unexpected fields (`stripUnknown` or equivalent)
  - Enforce type, format, length, range, and custom rules

### 8.4 Input Sanitization

- Inputs that allow free text (e.g., names, comments) should be sanitized to remove or neutralize dangerous content,
  especially when output is rendered in HTML or stored.
- HTML, script, or control characters should be escaped or stripped when output to a user.

### 8.5 Deny by Default

- If an input field is not explicitly allowed or documented, reject it.
- Unexpected parameters, keys, or fields must not be silently ignored. They should be either removed or rejected with a
  validation error.

---

## 9. Secure Error Handling

### 9.1 General Principles

- Error handling must never expose internal implementation details (e.g., stack traces, exception messages, file paths)
  to clients.
- All unexpected errors must be logged with sufficient context for developers to investigate, but responses to clients
  must be sanitized.

### 9.2 Client-Facing Error Responses

- Responses sent to the client must be generic and user-friendly:
  - Example: "An unexpected error occurred. Please try again later."
- Include a trace ID or request ID in the response for debugging purposes:
  - Example: "An error occurred. Reference ID: 9fd81234ab."
- Map known errors to appropriate HTTP status codes:
  - 400 Bad Request: for input validation failures
  - 401 Unauthorized / 403 Forbidden: for auth-related issues
  - 404 Not Found: for unknown resources
  - 500 Internal Server Error: for unexpected system errors

### 9.3 Exception Handling

- Catch and handle all unhandled exceptions at the framework or middleware level.
- Do not propagate raw exceptions to the response layer; convert to a controlled error object with safe messaging.
- Wrap known failure points (e.g., database, API calls, parsing) in try-catch blocks with custom error messages.

### 9.4 Secure Defaults

- In production mode:
  - Disable detailed error pages or debugging messages.
  - Disable framework-level default error handlers that leak stack traces.
- In development mode:
  - Allow verbose error messages only on localhost or authenticated debug interfaces.

---

## 10. Logging and Monitoring Specification

### 10.1 General Requirements

- All services must implement structured logging and distributed tracing when possible.
- Logs and traces should be correlated using a consistent trace ID or request ID across services.
- Logging and tracing systems should integrate with a central observability stack (e.g., ELK, Datadog, Grafana,
  OpenTelemetry, AWS X-Ray).

### 10.2 Secure Logging Practices

- Never log sensitive information, including:
  - Passwords
  - Access tokens, API keys, secrets
  - Credit card or PII (e.g., email addresses, phone numbers, national IDs)
- Apply filters/middleware to sanitize logs before they are written.

### 10.3 What to Log

Each log entry should include the following contextual metadata:

- Timestamp
- User ID (if authenticated)
- Request path and method
- Service name and component/module
- Action type (e.g., `user.login`, `invoice.create`)
- Trace ID / Correlation ID
- Status (success/failure) and relevant error messages (sanitized)

### 10.4 Tracing and Correlation

- A trace ID or request ID must be generated at the entry point (e.g., API Gateway, load balancer) and propagated
  through all downstream services.
- All logs during the lifecycle of a request must include the trace ID for easy correlation.
- Use tracing tools (e.g., OpenTelemetry, Jaeger, Zipkin) to capture spans across services and measure performance
  bottlenecks.

### 10.5 Log Levels

Use appropriate log levels to control verbosity and relevance:

- **DEBUG:** Diagnostic information during development/testing
- **INFO:** Expected runtime events (e.g., service started, user signed in)
- **WARN:** Unexpected events that don't interrupt flow (e.g., retrying after timeout)
- **ERROR:** Recoverable application failures
- **FATAL/CRITICAL:** Unrecoverable errors or system crashes

### 10.6 Log Retention and Storage

- Logs should be retained for a period based on compliance and business needs (e.g., 30–90 days).
- Logs must be stored securely with:
  - Access control
  - Encryption at rest and in transit
- Monitor log ingestion volumes to avoid storage overrun and unexpected costs.

### 10.7 Alerting and Monitoring

- Define alerts on critical events such as:
  - Failed login spikes
  - Unauthorized access attempts
  - API error rate thresholds
  - Service outages or abnormal latency
- Integrate alerting tools (e.g., PagerDuty, Opsgenie, Slack, GChat) to notify engineering teams in real time.

---

## 11. Password Management Specification

### 11.1 General Principles

- Passwords and other sensitive user data must never be stored in plaintext.
- If the data does not need to be retrieved or displayed (e.g., passwords), it must be hashed.
- If the data needs to be retrieved (e.g., personal identifiers, tokens), it must be encrypted.

### 11.2 Password Storage

- Passwords must be hashed using a modern, secure algorithm:
  - bcrypt (preferred)
  - argon2 (recommended for high-security applications)
  - PBKDF2 (acceptable in legacy systems)
- Do not use general-purpose hash functions like SHA-256, MD5, or SHA-1 for password storage.
- Always use a unique, cryptographically secure salt for each password to defend against rainbow table attacks.
- Set an appropriate cost/work factor to balance security and performance (e.g., bcrypt cost factor ≥ 12).

### 11.3 Password Verification

- During authentication, compare the hashed version of the user-provided password with the stored hash.
- Use a constant-time comparison to prevent timing attacks.

### 11.4 Other Sensitive Data

- Any sensitive data that needs to be retrieved (e.g., email, personal identification numbers, API keys) must be
  encrypted using:
  - ⚠ *Overridden by security.md (AES-256-GCM only):* AES-256-GCM or AES-256-CBC with HMAC for authenticated
    encryption
- Encryption keys:
  - Must be stored securely in a Key Management System (KMS) (e.g., AWS KMS, HashiCorp Vault, Azure Key Vault)
  - Must be rotated periodically and on suspected compromise

### 11.5 Access Controls and Auditing

- Only authorized systems or services should have access to decrypt sensitive data.
- Implement fine-grained access controls and audit logging for all encryption/decryption operations.

### 11.6 Logging

- Never log passwords or decrypted sensitive data.
- Ensure application logs are sanitized and reviewed regularly.

---

## 12. Session Management Specification

### 12.1 General Principles

- Session data must be stored securely, isolated from the main application logic and user-accessible areas.
- Sessions must never be stored in the file system using raw or insecure serialization formats (e.g., PHP, Python, Ruby
  serialized objects) without strict access controls.

### 12.2 Session Storage

- Sessions must be stored in a secure, centralized session store, such as:
  - Redis
  - Memcached
  - Database-backed stores (e.g., PostgreSQL, MySQL with dedicated session tables)
- File-based session storage must be avoided, especially if session data is serialized. If unavoidable:
  - Enforce strict file permissions (read/write only by the application)
  - Disallow direct user access to session directories
  - Use secure serialization formats (e.g., JSON), never language-native ones prone to deserialization
    vulnerabilities
  - Validate and sanitize all session data before usage

### 12.3 Token-Based Sessions (Recommended)

- Prefer stateless session management using secure tokens (e.g., JWTs) where feasible:
  - Tokens must be signed (e.g., RS256, HS256) and optionally encrypted
  - Use short-lived access tokens with optional refresh tokens
  - Store tokens in HttpOnly, Secure, SameSite=strict cookies, or in secure memory (not localStorage)

### 12.4 Session Integrity and Tamper Protection

- Session data must not be editable by the user.
- If using cookie-based sessions, they must be signed to prevent tampering.
- All session modifications must be performed server-side and validated against expected schemas.

### 12.5 Expiration and Invalidation

- All sessions must have an expiration time (e.g., 15–30 minutes of inactivity).
- Support manual session invalidation (e.g., logout, password reset).
- Support invalidation of all sessions in response to:
  - Account compromise
  - MFA change
  - Role/permission changes

### 12.6 Secure Attributes

- If sessions are stored in cookies, the following cookie attributes must be set:
  - `HttpOnly`: to prevent access via JavaScript
  - `Secure`: to ensure transmission only over HTTPS
  - `SameSite=Strict` or `Lax`: to prevent CSRF

### 12.7 Session Hijacking and Replay Protection

- Implement protections against session hijacking:
  - Regenerate session IDs on login and privilege elevation
  - ⚠ *Overridden by security.md (never bind a session to an IP address):* Monitor IP/User-Agent consistency; bind
    the session with an IP address
  - Invalidate sessions after suspicious activity
- Log and monitor failed or unusual session access attempts.

---

## 13. Developer & Team Security Practices

### 13.1 Security Training & Awareness

- Conduct periodic security awareness workshops, code review sessions, and red team simulations.
- Integrate security checklists and cheat sheets into development workflows.

### 13.2 Code Signing & Verification

- When possible sign the code artifacts before deployments.
- Use tools such as:
  - Sigstore / Cosign for container image signing
  - GPG or SLSA framework for build artifact verification
- Verify signatures during CI/CD to prevent supply chain tampering.

### 13.3 Static and Dynamic Code Analysis

- Integrate automated code scanning tools into the CI/CD pipeline:
  - **SAST (Static Application Security Testing):**
    - Run on every pull request or pre-merge
    - Detects code-level issues like hardcoded secrets, insecure function usage
    - Tools: SonarQube, Semgrep, CodeQL
  - **DAST (Dynamic Application Security Testing):**
    - Run against deployed environments or test instances
    - Detects runtime vulnerabilities like XSS, SQL injection, etc.
    - Tools: OWASP ZAP, Burp Suite, Detectify
  - **IAST (Interactive Application Security Testing):**
    - Combines SAST + DAST + runtime analysis for deeper insights
    - Tools: Contrast Security, Seeker

### 13.4 Dependency & Supply Chain Scanning

- When applicable use dependency scanning tools to ensure dependencies are up to date.
  - GitHub Dependabot, Snyk, OSS Review Toolkit, Trivy, or similar
- Set up policies to:
  - Block builds on high/critical CVEs
  - Auto-update dependencies if safe patches are available
- Maintain SBOM (Software Bill of Materials) for all released software.

### 13.5 Security in CI/CD

- Implement security checks as first-class citizens in the CI/CD pipeline:
  - Linting, SAST, DAST, secret scanning, license checks
  - Enforce review gates for high-risk changes
- Restrict write access to CI/CD secrets and environments.
- Rotate CI/CD credentials and tokens regularly.

### 13.6 Patch & Vulnerability Management

- Regularly monitor vulnerability disclosures (CVE feeds, vendor advisories).
- Establish a defined patching SLA (the following is a recommended SLA for security issues):
  - Critical: 24–72 hours
  - High: within 1 week
- Track dependency and base image versions.
- Automate container image updates using tools like Renovate or Kustomize with GitOps.

### 13.7 Secure Development Lifecycle (SDLC)

- Embed security into all stages of development:
  - **Design:** Threat modeling, secure architecture reviews
  - **Development:** Secure coding standards, code reviews
  - **Testing:** Automated security scans, manual pen testing
  - **Deployment:** Immutable infrastructure, signed artifacts
- Include security checklists in sprint definitions and DoD (Definition of Done).

### 13.8 Secrets Management

- Never hardcode secrets, credentials, or tokens in source code or config files.
- Use dedicated secret management tools:
  - HashiCorp Vault
  - AWS Secrets Manager
  - Azure Key Vault
- Limit access to secrets based on the principle of least privilege.
- Audit and rotate secrets regularly.

---

## 14. Database Security Specification

### 14.1 General Principles

- Databases must never be directly exposed to the public internet.
- All database access must be routed through application services, secure bastion hosts or use a zero trust network
  solution.
- Apply the principle of least privilege to all database users and service accounts.

### 14.2 Database User Privileges

- Applications must not connect to the database using high-privilege accounts (e.g., root, postgres, admin, sa).
- Create dedicated database users for each application or service with only the permissions required for their
  function:
  - Read-only users for reporting or analytics services
  - Read-write users for application services (limited to specific tables/schemas)
  - Administrative users only for migrations, schema changes, and maintenance tasks (never used at runtime)
- Avoid granting the following privileges to application users:
  - `DROP`, `TRUNCATE`, or `ALTER` on production tables
  - `GRANT` or `SUPERUSER` privileges
  - Access to system catalogs or administrative functions

### 14.3 Network Isolation

- Databases must be deployed in private subnets with no direct internet access.
- Access must be restricted using:
  - Security groups or firewall rules allowing only authorized application servers
  - VPC peering or private endpoints for cross-service access
- If remote access is required for administrative purposes:
  - Use a bastion host or VPN with MFA
  - Use SSH tunneling or secure connection proxies
  - Never expose database ports (e.g., 3306, 5432, 27017) to `0.0.0.0/0`

### 14.4 Secure Public Access (When Required)

- If a database must be accessed from external sources (e.g., third-party integrations, partner APIs):
  - Use a read-only replica with restricted access
  - Expose access through a secure API layer, not direct database connections
  - Implement IP allowlisting for known external sources
  - Use SSL/TLS for all connections (enforce `sslmode=require` or equivalent)
  - Consider using a database proxy (e.g., PgBouncer, ProxySQL, AWS RDS Proxy) to manage connections securely

### 14.5 Connection Security

- All database connections must use TLS/SSL encryption.
- Enforce encrypted connections at the database level (reject unencrypted connections).
- Use certificate-based authentication where supported.
- Rotate database credentials periodically and on suspected compromise.

### 14.6 Query Safety

- Use parameterized queries or prepared statements to prevent SQL injection.
- Never construct SQL queries using string concatenation with user input.
- Use ORM-provided query builders with proper escaping.
- Implement query timeouts to prevent long-running queries from affecting performance.

### 14.7 Auditing and Monitoring

- Enable database audit logging for:
  - Authentication attempts (successful and failed)
  - Schema changes (DDL statements)
  - Privileged operations
  - Access to sensitive tables
- Integrate database logs with the centralized logging and monitoring stack.
- Alert on:
  - Failed authentication attempts
  - Unusual query patterns or volumes
  - Connections from unexpected IP addresses

### 14.8 Backup and Recovery

- Automate regular database backups with encryption at rest.
- Store backups in a separate, secure location with restricted access.
- Test backup restoration procedures periodically.
- Implement point-in-time recovery (PITR) where supported.
