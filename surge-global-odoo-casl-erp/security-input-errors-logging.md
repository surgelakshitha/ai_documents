---
keywords: input, inputs, validation, validate, schema, payload, body, webhook, websocket, regex, mass assignment, deserialize, error, errors, exception, exceptions, stack trace, traceback, logging, logger, log, logs, trace, redact, pii, audit, api_action
always: false
---

# Security: input validation, errors and logging

*Part of SECURITY.MD, split by topic (start file: `security.md`). § numbers are those of SECURITY.MD. Binding.*

## 8. Input Validation

- All input is untrusted: bodies, query params, path params, headers, cookies, WebSocket frames, webhook payloads, queue messages, and third-party API responses.
- Validate at the boundary with Zod (preferred in this stack), before any business logic:

```js
const Body = z.object({
  invoiceId: z.string().uuid(),
  amount: z.number().int().positive().max(1_000_000),
  status: z.enum(['draft', 'sent', 'paid']),
}).strict(); // .strict() rejects unknown keys — required
```

- `.strict()` / `stripUnknown` is mandatory. Silently ignoring unexpected fields is how mass-assignment bugs ship.
- Validate type, format, length, range, and enum membership. Reject; do not coerce.
- NEVER pass a request body straight into an ORM create/update. Explicitly pick the allowed fields.
- **ReDoS:** no user input against a regex with nested quantifiers or ambiguous alternation. Prefer parsers and length caps. Cap input length before the regex runs.
- **Prototype pollution:** never deep-merge untrusted objects into config/defaults. Reject payloads containing `__proto__`, `constructor`, `prototype` keys. Use `Object.create(null)` for lookup maps.
- NEVER deserialize untrusted data into language-native objects. JSON only.
- Validate outbound too: a redirect target, a filename, or a webhook URL derived from user input needs the same treatment (open redirect, path traversal, SSRF).

## 9. Error Handling

- Client responses are generic + a trace ID: `{"error":"Something went wrong","traceId":"9fd81234ab"}`.
- Full detail (stack, cause, query) goes to the log, correlated by the same trace ID.
- Status mapping: `400` validation, `401` unauthenticated, `403` unauthorized, `404` not found, `409` conflict, `429` rate limited, `500` unexpected.
- Do not leak existence through status codes. For a resource the caller may not access, return `404`, not `403` — `403` confirms it exists.
- Catch at framework/middleware level. Never let a raw exception reach the response serializer. Never `catch (e) { }`.
- Production: verbose errors and framework debug handlers off. Verify this is enforced by config, not by remembering.
- Error messages NEVER echo the input that caused them if that input may contain secrets.

## 10. Logging & Monitoring

- Structured JSON logs. Every entry: timestamp, level, service, trace ID, tenant ID, user ID (if any), action, outcome.
- Trace ID generated at the edge and propagated through every downstream call (OpenTelemetry).
- **Redaction is a middleware, not a discipline.** Implement a serializer that drops/masks known-sensitive keys (`password`, `token`, `authorization`, `cookie`, `secret`, `apiKey`, `card`, `ssn`, `nic`) and mask P1/P2 data. Do not rely on developers remembering.
- NEVER log: credentials, tokens, session IDs, full request bodies of auth endpoints, decrypted sensitive data, full PII.
- Log security events explicitly: login success/failure, MFA challenge/result, permission denial, tenant switch, admin action, password/email change, rate-limit trip.
- Logs encrypted at rest and in transit, access-controlled, retained per policy (30–90 days baseline).
- Alerts on: failed-login spikes, authorization-denial spikes, blocked-SSRF attempts, error-rate thresholds, unusual query volume, connections from unexpected IPs.
