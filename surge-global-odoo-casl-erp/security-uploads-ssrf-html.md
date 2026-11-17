---
keywords: upload, uploads, url, urls, fetch, ssrf, image, images, file, attachment, attachments, filename, path traversal, svg, html, xss, sanitize, sanitise, markdown, wysiwyg, innerhtml, csp, pre-signed, presigned, redirect, requests
always: false
---

# Security: untrusted URLs, uploads and HTML

*Part of SECURITY.MD, split by topic (start file: `security.md`). § numbers are those of SECURITY.MD. Binding.*

## 6. Untrusted URLs, SSRF, and Uploads

### 6.1 Default position

**Do not fetch user-supplied URLs server-side.** Use pre-signed direct-to-storage uploads (§6.3). This removes the vulnerability class rather than mitigating it. If asked to implement server-side URL fetching, propose pre-signed uploads first and explain why.

### 6.2 If server-side fetching is unavoidable

The source guideline's "issue a HEAD, then GET if it returns 200" is **not a security control** and this file overrides it. The HEAD request is itself an SSRF, and the target can answer HEAD from a benign host then serve the GET from an internal one (DNS rebinding / redirect). Do not rely on it.

Required controls, all of them:

1. **Isolate the fetcher.** Separate Lambda/container, in a subnet with no route to internal services, IMDS disabled (or IMDSv2 with hop limit 1), and an egress allowlist. This is the control that actually works; the rest is defense-in-depth.
2. Parse and constrain the URL: scheme in `{https}` only. No `http`, `file`, `gopher`, `ftp`, `data`, `blob`.
3. **Resolve DNS yourself, validate every resolved address, then connect to that pinned IP** with the `Host` header set. Re-validating after resolution is what closes the rebinding window.
4. Block, for every resolved address (IPv4 and IPv6): `0.0.0.0/8`, `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16` (includes `169.254.169.254`), `100.64.0.0/10`, `192.0.0.0/24`, `198.18.0.0/15`, multicast, reserved; `::1`, `fc00::/7`, `fe80::/10`, and IPv4-mapped forms (`::ffff:127.0.0.1`).
5. Disable redirect following, or re-run steps 3–4 on every hop, max 3 hops.
6. Timeouts (connect + total, e.g. 3s/10s), max response size enforced while streaming (not from `Content-Length`, which lies), and a hard cap on concurrent fetches.
7. Validate content by magic bytes, not by extension or `Content-Type`.
8. Log source user, tenant, target URL, resolved IP, and outcome. Alert on repeated attempts against blocked ranges — that is an attacker probing, not a bug.

> **Note:** the library `http-acl` referenced in the source doc is not a drop-in for Node — verify the ecosystem before adopting it. Prefer explicit resolve-and-validate with an IP-range library you have reviewed. *(Unverified — confirm before use.)*

### 6.3 Pre-signed uploads (preferred path)

- Server authenticates and authorizes the user, then issues a pre-signed URL: expiry 5–15 min, single-use where supported, content-type constrained to the image MIME allowlist, max size enforced at the storage policy level (not just in the client).
- Dedicated bucket, isolated from application data, private ACL by default, public access block on.
- Object keys are server-generated (`{tenantId}/{uuid}`) — NEVER derived from the client-supplied filename. Filenames are a path-traversal vector.
- Post-upload, before the object is usable: verify magic bytes, re-check size, re-encode the image to strip metadata and embedded payloads, optionally malware-scan.
- Serve through a CDN or app-controlled endpoint. Never expose raw storage URLs. Direct access uses short-lived pre-signed download URLs.
- Log and rate-limit pre-signed URL generation per user. Excessive generation is abuse.

### 6.4 Non-image files

SVG is not a safe image — it executes script. Either exclude it from the allowlist or sanitize it as HTML (§7) and serve it from a separate origin with `Content-Disposition: attachment`. Any other file type requires human review.

## 7. Rendering Untrusted HTML

- Sanitize with DOMPurify at render time, with an **explicit allowlist** of tags and attributes. Never a denylist.
- Forbidden regardless of allowlist: `<script>`, `<iframe>`, `<object>`, `<embed>`, `<style>`, `<form>`, `<base>`, all `on*` handlers, `javascript:` / `data:` URLs in `href` / `src` / `srcset`.
- Store raw input (markdown or plain text). NEVER store pre-rendered HTML — it freezes a sanitizer version into your database and every future bypass becomes permanent.
- Treat stored content and reflected content identically. Any API returning user content is a stored-XSS vector.
- Applies everywhere: `dangerouslySetInnerHTML`, `.innerHTML`, `v-html`, markdown→HTML pipelines, WYSIWYG output, email templates, PDF generation.
- On SSR: sanitize server-side too (DOMPurify + jsdom). SSR fetching of `<img src="http://169.254.169.254/...">` is SSRF (§6).
- Ship a strict CSP as defense-in-depth — it is not a substitute for sanitization: `default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'`
- No `'unsafe-inline'` / `'unsafe-eval'` in `script-src`. If you need inline scripts, use nonces or hashes.
- NEVER interpolate untrusted data into a `<script>` block, a URL, or a CSS value. Context-correct encoding, or don't do it.
