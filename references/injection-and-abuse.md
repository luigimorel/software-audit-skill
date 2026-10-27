# Injection, business logic, abuse, and AI features

Read this when auditing input handling, checkout and payment flows, rate limits, uploads, SSRF, webhooks, and LLM-powered features.

## Contents

- Injection (OWASP A03)
- Business logic, abuse, and cost (the vibe-coder special)
- AI-feature specifics (if the app uses LLMs)
- Input validation at the boundary
- Sources

Rate limiting and billing protection have their own file: `rate-limiting.md`.

---

## Injection (OWASP A03)

**Look for:**

- SQL built with string concatenation or template literals (`` `SELECT * FROM users WHERE id = ${id}` ``), including inside ORM raw-query escape hatches (`$queryRawUnsafe`, `db.raw`, `Exec(fmt.Sprintf(...))`)
- NoSQL operator injection (passing `req.body` directly into Mongo filters)
- ORM operator injection: `prisma.user.findFirst({ where: req.body })` lets a caller send `{ "email": { "contains": "" } }` and match every row. Validate the shape first, then build the filter yourself. `$queryRawUnsafe` / `$executeRawUnsafe` with user input bypass parameterization entirely; use the tagged-template `$queryRaw` instead.
- Command injection (`exec`, `child_process`, `os/exec` with user input in a shell string)
- XSS: `dangerouslySetInnerHTML`, `v-html`, `innerHTML`, `document.write`, unsanitized Markdown/HTML rendering, user content in `<script>` or inline handlers, `href={userInput}` allowing `javascript:` URLs
- Template injection, path traversal (`../` in file paths built from user input), open redirects (`redirect(req.query.next)`)
- LLM prompt injection where user or web content flows into a prompt that can call tools or see other users' data

**Fix pattern:** parameterized queries, allow-list validation (zod, valibot, go-playground/validator), contextual output encoding, sanitize with DOMPurify when HTML is truly needed, avoid shelling out, validate redirect targets against an allow-list.

## Business logic, abuse, and cost (the vibe-coder special)

- **Client-trusted prices, quantities, discounts, or plan tiers** (checkout totals computed in the browser, entitlement flags in the client); verify payments and webhooks server-side with signature checks
- Race conditions on balance, coupon, inventory, or "one per user" logic
- Missing rate limits and quotas on expensive endpoints: AI/LLM calls, email/SMS sending, file processing, image generation, search. An open endpoint means an open wallet.
- Unauthenticated endpoints that call paid APIs (the leaked-key problem in another form)
- Upload endpoints: no type, size, or content check; storing in a web-served path; SVG/HTML uploads executing scripts; path traversal in filenames
- SSRF (OWASP A10): server fetching user-supplied URLs (link previews, webhooks, image import) without blocking internal/metadata IPs like `169.254.169.254`
- Webhooks accepted without signature verification; replayable
- Email features that can be abused for spam or phishing (arbitrary recipient/body)

## AI-feature specifics (if the app uses LLMs)

- Provider keys in client code; no per-user usage caps
- User content or retrieved documents placed in prompts that can trigger tools, DB queries, or actions without confirmation
- System prompts containing secrets or internal URLs
- Model output rendered as HTML or executed (XSS, command execution)
- RAG that ignores per-user permissions (user A's documents retrievable by user B)
- Sensitive user data sent to third-party models without disclosure or opt-out
- Chat transcripts stored and logged in full with no retention policy
- Untrusted text concatenated into the system prompt instead of sent as a separate `user` message (this lowers, but does not remove, prompt-injection risk)
- Tool or function calling without a safe allow-list, schema validation of the model's parameters, least-privilege access (read-only where possible), or an audit log of invocations; never let the model assemble raw SQL or shell commands from user text
- No hard spending cap set at the provider, and no per-user token or request budget in the app (provider caps can lag)

## Input validation at the boundary

TypeScript types disappear at runtime, so a malformed request sails past them. Validate every external input with a runtime schema (Zod, Valibot, Yup, Joi, go-playground/validator) at: route handlers, Server Actions, webhook handlers, form submissions, and URL or query parameters. Then copy only the allowed fields into DB writes (mass assignment guard):

```ts
// Bad: caller can add { isAdmin: true, credits: 99999 }
await db.users.update({ where: { id }, data: req.body });

// Good
const { name, email } = CreateUserSchema.parse(req.body);
await db.users.update({ where: { id }, data: { name, email } });
```

## Sources

- [OWASP Top 10 A03: Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Prisma: raw queries and SQL injection](https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access/raw-queries)
